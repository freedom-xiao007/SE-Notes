# Soul网关源码解析（九）插件配置加载初探
***
## 简介
&ensp;&ensp;&ensp;&ensp;今日来探索一下插件的初始化，及相关的配置的加载

## 源码Debug
### 插件初始化
&ensp;&ensp;&ensp;&ensp;首先来到我们非常熟悉的插件链调用的类： SoulWebHandler ,在其中的 DefaultSoulPluginChain ,我们看到plugins是通过构造函数传入的

&ensp;&ensp;&ensp;&ensp;我们来跟踪一下是谁传入的，在SoulWebHandler的public那一行打上断点，重启后跟踪调用栈

```java
public final class SoulWebHandler implements WebHandler {

    private final List<SoulPlugin> plugins;

    // 在这里也看到plugins是通过构造函数传入的，在public这一行打上断点，然后重启
    public SoulWebHandler(final List<SoulPlugin> plugins) {
        this.plugins = plugins;
        String schedulerType = System.getProperty("soul.scheduler.type", "fixed");
        if (Objects.equals(schedulerType, "fixed")) {
            int threads = Integer.parseInt(System.getProperty(
                    "soul.work.threads", "" + Math.max((Runtime.getRuntime().availableProcessors() << 1) + 1, 16)));
            scheduler = Schedulers.newParallel("soul-work-threads", threads);
        } else {
            scheduler = Schedulers.elastic();
        }
    }

    @Override
    public Mono<Void> handle(@NonNull final ServerWebExchange exchange) {
        MetricsTrackerFacade.getInstance().counterInc(MetricsLabelEnum.REQUEST_TOTAL.getName());
        Optional<HistogramMetricsTrackerDelegate> startTimer = MetricsTrackerFacade.getInstance().histogramStartTimer(MetricsLabelEnum.REQUEST_LATENCY.getName());
        // 传入 plugins 到 DefaultSoulPluginChain
        return new DefaultSoulPluginChain(plugins).execute(exchange).subscribeOn(scheduler)
                .doOnSuccess(t -> startTimer.ifPresent(time -> MetricsTrackerFacade.getInstance().histogramObserveDuration(time)));
    }

    private static class DefaultSoulPluginChain implements SoulPluginChain {

        private int index;

        private final List<SoulPlugin> plugins;

        // 通过构造函数得到 plugins
        DefaultSoulPluginChain(final List<SoulPlugin> plugins) {
            this.plugins = plugins;
        }

        @Override
        public Mono<Void> execute(final ServerWebExchange exchange) {
            return Mono.defer(() -> {
                if (this.index < plugins.size()) {
                    SoulPlugin plugin = plugins.get(this.index++);
                    Boolean skip = plugin.skip(exchange);
                    if (skip) {
                        return this.execute(exchange);
                    }
                    return plugin.execute(exchange, this);
                }
                return Mono.empty();
            });
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;我这有点慢，重启等以后后，我们查看上一个调用，来到了 SoulConfiguration

&ensp;&ensp;&ensp;&ensp;通过类的大致信息可以看到是Spring之类的加载机制。通过前面文章的分析中，大致应该就是配置了 auto configuration ，然后启动的时候Spring进行加载的

```java
public class SoulConfiguration {

    @Bean("webHandler")
    public SoulWebHandler soulWebHandler(final ObjectProvider<List<SoulPlugin>> plugins) {
        List<SoulPlugin> pluginList = plugins.getIfAvailable(Collections::emptyList);
        final List<SoulPlugin> soulPlugins = pluginList.stream()
                .sorted(Comparator.comparingInt(SoulPlugin::getOrder)).collect(Collectors.toList());
        soulPlugins.forEach(soulPlugin -> log.info("load plugin:[{}] [{}]", soulPlugin.named(), soulPlugin.getClass().getName()));
        return new SoulWebHandler(soulPlugins);
    }
}
```

&ensp;&ensp;&ensp;&ensp;大致就知道是Spring自动加载的，通过AutoConfiguration机制，下面就是Spring相关了，这里就不继续跟踪了

### 配置初始化
&ensp;&ensp;&ensp;&ensp;在上一篇的文章分析中，我们分析了路由匹配的大致细节，需要选择器和规则的配置，下面我们来跟踪一下这些配置的初始化

&ensp;&ensp;&ensp;&ensp;首先进入路由匹配核心类： AbstractSoulPlugin ，其中进行 pluginData 、selectorData、rules的获取，如下面代码注释：

```java
public abstract class AbstractSoulPlugin implements SoulPlugin {

    @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        String pluginName = named();
        // 获取插件数据
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
        if (pluginData != null && pluginData.getEnabled()) {
            // 根据插件名称，获取选择器数据
            final Collection<SelectorData> selectors = BaseDataCache.getInstance().obtainSelectorData(pluginName);
            if (CollectionUtils.isEmpty(selectors)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            final SelectorData selectorData = matchSelector(exchange, selectors);
            if (Objects.isNull(selectorData)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            selectorLog(selectorData, pluginName);
            // 根据选择器的id，获取其规则数据
            final List<RuleData> rules = BaseDataCache.getInstance().obtainRuleData(selectorData.getId());
            if (CollectionUtils.isEmpty(rules)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            RuleData rule;
            if (selectorData.getType() == SelectorTypeEnum.FULL_FLOW.getCode()) {
                rule = rules.get(rules.size() - 1);
            } else {
                rule = matchRule(exchange, rules);
            }
            if (Objects.isNull(rule)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            ruleLog(rule, pluginName);
            return doExecute(exchange, chain, selectorData, rule);
        }
        return chain.execute(exchange);
    }
}
```

&ensp;&ensp;&ensp;&ensp;我们到这个类： BaseDataCache 看一下，看到它有两个数据的Map存储：插件数据、选择器数据、规则数据

```java
public final class BaseDataCache {
     
    /**
     * pluginName -> PluginData.
     */
    private static final ConcurrentMap<String, PluginData> PLUGIN_MAP = Maps.newConcurrentMap();
    
    /**
     * pluginName -> SelectorData.
     */
    private static final ConcurrentMap<String, List<SelectorData>> SELECTOR_MAP = Maps.newConcurrentMap();
    
    /**
     * selectorId -> RuleData.
     */
    private static final ConcurrentMap<String, List<RuleData>> RULE_MAP = Maps.newConcurrentMap();
    
}
```

&ensp;&ensp;&ensp;&ensp;我们发现它是一个静态单例，而插件数据的配置是使用下面的函数，我们在函数上打上断点，看看是谁调用了它，它的调用栈是怎么样的

```java
public final class BaseDataCache {
    public void cachePluginData(final PluginData pluginData) {
        Optional.ofNullable(pluginData).ifPresent(data -> PLUGIN_MAP.put(data.getName(), data));
    }
}
```

&ensp;&ensp;&ensp;&ensp;在public这一行打上断点，重启，顺利进入上面的函数，我们查看下 pluginData ，还挺简单，大致如下

```
PluginData(id=1, name=sign, config=null, role=1, enabled=false)
```

&ensp;&ensp;&ensp;&ensp;我们跟踪其调用栈，其调用链的类和函数如下，类在调用栈的顺序也是对应的

```java
public class CommonPluginDataSubscriber implements PluginDataSubscriber {
    
    // 在这个里面触发 subscribeDataHandler , subscribeDataHandler 触发 DaseDataCache的调用
    @Override
    public void onSubscribe(final PluginData pluginData) {
        subscribeDataHandler(pluginData, DataEventTypeEnum.UPDATE);
    }
    
    private <T> void subscribeDataHandler(final T classData, final DataEventTypeEnum dataType) {
        Optional.ofNullable(classData).ifPresent(data -> {
            // 插件数据的操作
            if (data instanceof PluginData) {
                PluginData pluginData = (PluginData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    // 在这进行触发调用
                    BaseDataCache.getInstance().cachePluginData(pluginData);
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.handlerPlugin(pluginData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    BaseDataCache.getInstance().removePluginData(pluginData);
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.removePlugin(pluginData));
                }
            // 选择器的操作
            } else if (data instanceof SelectorData) {
                SelectorData selectorData = (SelectorData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    BaseDataCache.getInstance().cacheSelectData(selectorData);
                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.handlerSelector(selectorData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    BaseDataCache.getInstance().removeSelectData(selectorData);
                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.removeSelector(selectorData));
                }
            // 规则的操作
            } else if (data instanceof RuleData) {
                RuleData ruleData = (RuleData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    BaseDataCache.getInstance().cacheRuleData(ruleData);
                    Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.handlerRule(ruleData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    BaseDataCache.getInstance().removeRuleData(ruleData);
                    Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.removeRule(ruleData));
                }
            }
        });
    }
}

// 这个类中触发了上面单个插件的配置，并且这个类拿到了所有的插件数据
public class PluginDataHandler extends AbstractDataHandler<PluginData> {

    @Override
    protected void doRefresh(final List<PluginData> dataList) {
        // 触发上面的调用
        pluginDataSubscriber.refreshPluginDataSelf(dataList);
        dataList.forEach(pluginDataSubscriber::onSubscribe);
    }
}

// 继续跟下去，看看datalist的来源
public abstract class AbstractDataHandler<T> implements DataHandler {

    @Override
    public void handle(final String json, final String eventType) {
        List<T> dataList = convert(json);
        if (CollectionUtils.isNotEmpty(dataList)) {
            DataEventTypeEnum eventTypeEnum = DataEventTypeEnum.acquireByName(eventType);
            switch (eventTypeEnum) {
                case REFRESH:
                case MYSELF:
                    // 触发上面函数的调用
                    doRefresh(dataList);
                    break;
                case UPDATE:
                case CREATE:
                    doUpdate(dataList);
                    break;
                case DELETE:
                    doDelete(dataList);
                    break;
                default:
                    break;
            }
        }
    }
}

// 来到了websocket，进行跟
public class WebsocketDataHandler {
    public void executor(final ConfigGroupEnum type, final String json, final String eventType) {
        ENUM_MAP.get(type).handle(json, eventType);
    }
}

// 在这个类中，看到插件数据列表是从websocket来的，那就是从Soul-Admin那边获取来到的数据
public final class SoulWebsocketClient extends WebSocketClient {

    public void onMessage(final String result) {
        handleResult(result);
    }  

    @SuppressWarnings("ALL")
    private void handleResult(final String result) {
        WebsocketData websocketData = GsonUtils.getInstance().fromJson(result, WebsocketData.class);
        ConfigGroupEnum groupEnum = ConfigGroupEnum.acquireByName(websocketData.getGroupType());
        String eventType = websocketData.getEventType();
        String json = GsonUtils.getInstance().toJson(websocketData.getData());
        websocketDataHandler.executor(groupEnum, json, eventType);
    }
}
```

&ensp;&ensp;&ensp;&ensp;看下result的大致内容：也就是一些简单的配置

```json
{
	"groupType": "PLUGIN",
	"eventType": "MYSELF",
	"data": [{
		"id": "1",
		"name": "sign",
		"role": 1,
		"enabled": false
	}, {
		"id": "9",
		"name": "hystrix",
		"role": 0,
		"enabled": false
	}]
}
```

&ensp;&ensp;&ensp;&ensp;结合路由匹配的关键函数,好像它的作用就是用来判断是否开启和根据name获得选择器

```java
public abstract class AbstractSoulPlugin implements SoulPlugin {

    @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        String pluginName = named();
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
        // 用于判断是否开启
        if (pluginData != null && pluginData.getEnabled()) {
            .......
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;插件数据这一块的流程定位下来比较轻松，毕竟前面流程也梳理差不多了

&ensp;&ensp;&ensp;&ensp;选择器和规则数据的初始化流程，这里就不进行定位了，逻辑和插件数据基本一样，这里就不赘述了

## 总结
&ensp;&ensp;&ensp;&ensp;今天分析了插件的初始化和相关配置的初始化

&ensp;&ensp;&ensp;&ensp;插件的初始化是使用Spring的自动配置加载机制进行实现了，结合使用插件都需要引入哪些start的依赖

&ensp;&ensp;&ensp;&ensp;插件数据、选择器数据、规则数据的初始化，都是从Websocket（或者同步通信模块）那边过来的，接收到数据以后，进行配置加载到本地

&ensp;&ensp;&ensp;&ensp;我们还看到一些删除和更新相关的操作，这些属于后面的数据同步分析了，这里先放一放

## Soul网关源码分析文章列表
### Github
- [Soul源码阅读（一） 概览](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB1-%E6%A6%82%E8%A7%88.md)
- [Soul源码阅读（二）代码初步运行](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB2-%E5%88%9D%E6%AD%A5%E8%BF%90%E8%A1%8C.md)
- [Soul源码阅读（三）HTTP请求处理概览](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB3-%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%A6%82%E8%A7%88.md)
- [Soul网关源码阅读（四）Dubbo请求概览](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB4-dubbo%E8%AF%B7%E6%B1%82%E6%A6%82%E8%A7%88.md)
- [Soul网关源码阅读（五）请求类型探索](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB5-%E8%AF%B7%E6%B1%82%E7%B1%BB%E5%9E%8B%E6%8E%A2%E7%B4%A2.md)
- [Soul网关源码阅读（六）Sofa请求处理概览](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB6-sofa%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%A6%82%E8%A7%88.md)
- [Soul网关源码阅读（七）限流插件初探](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB7-%E9%99%90%E6%B5%81%E6%8F%92%E4%BB%B6%E5%88%9D%E6%8E%A2.md)
- [Soul网关源码阅读（八）路由匹配初探](https://github.com/lw1243925457/SE-Notes/blob/0e6931519a84d5c603504b2c6a633698ac793b70/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB8-%E8%B7%AF%E7%94%B1%E5%8C%B9%E9%85%8D%E5%88%9D%E6%8E%A2.md)

- [Soul网关源码阅读番外篇（一） HTTP参数请求错误](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%95%AA%E5%A4%96%E7%AF%871-HTTP%E7%A4%BA%E4%BE%8B%E5%8F%82%E6%95%B0%E8%AF%B7%E6%B1%82%E9%94%99%E8%AF%AF.md)

### 掘金
- [Soul网关源码阅读（一） 概览](https://juejin.cn/post/6917864624423436296)
- [Soul网关源码阅读（二）代码初步运行](https://juejin.cn/post/6917865804121767944)
- [Soul网关源码阅读（三）请求处理概览](https://juejin.cn/post/6917866538712334343)
- [Soul网关源码阅读（四）Dubbo请求概览](https://juejin.cn/post/6917867369909977102)
- [Soul网关源码阅读（五）请求类型探索](https://juejin.cn/post/6918575905962983438)
- [Soul网关源码阅读（六）Sofa请求处理概览](https://juejin.cn/post/6918736260467015693)
- [Soul网关源码阅读（七）限流插件初探](https://juejin.cn/post/6919348164944232455/)
- [Soul网关源码阅读（八）路由匹配初探](https://juejin.cn/post/6919774553241550855/)

- [Soul网关源码阅读番外篇（一） HTTP参数请求错误](https://juejin.cn/post/6918947689564471309)