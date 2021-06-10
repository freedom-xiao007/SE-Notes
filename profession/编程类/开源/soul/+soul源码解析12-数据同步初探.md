# Soul网关源码解析（十二）数据同步初探-Bootstrap端
***
## 简介
&ensp;&ensp;&ensp;&ensp;此篇开始进入分析Soul网关的数据同步相关源码，首先看下数据是哪些数据，同步大致原理和方式

## 源码Debug
&ensp;&ensp;&ensp;&ensp;数据这里应该路由相关的数据，在程序中用来进行规则匹配、获取后端服务器真实地址之类的

&ensp;&ensp;&ensp;&ensp;在前面的处理流程的分析中，我们知道进行路由匹配的核心代码如下：

```java
public abstract class AbstractSoulPlugin implements SoulPlugin {
    ......
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        String pluginName = named();
        // 获取插件配置
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
        if (pluginData != null && pluginData.getEnabled()) {
            // 获取选择器
            final Collection<SelectorData> selectors = BaseDataCache.getInstance().obtainSelectorData(pluginName);
            .......
            // 获取规则
            final SelectorData selectorData = matchSelector(exchange, selectors);
            .......
        }
        return chain.execute(exchange);
    }
    ......
}
```

&ensp;&ensp;&ensp;&ensp;从上面可以看出数据大致有三种：插件配置数据、选择器数据、规则数据，这些数据都是从 BaseDataCache中获取的，我们来看看其核心代码：

```java
public final class BaseDataCache {
    
    private static final BaseDataCache INSTANCE = new BaseDataCache();
    
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

    public void cachePluginData(final PluginData pluginData) {
        Optional.ofNullable(pluginData).ifPresent(data -> PLUGIN_MAP.put(data.getName(), data));
    }

    public void removePluginData(final PluginData pluginData) {
        Optional.ofNullable(pluginData).ifPresent(data -> PLUGIN_MAP.remove(data.getName()));
    }
    
    public PluginData obtainPluginData(final String pluginName) {
        return PLUGIN_MAP.get(pluginName);
    }

    public void cacheSelectData(final SelectorData selectorData) {
        Optional.ofNullable(selectorData).ifPresent(this::selectorAccept);
    }

    public void removeSelectData(final SelectorData selectorData) {
        Optional.ofNullable(selectorData).ifPresent(data -> {
            final List<SelectorData> selectorDataList = SELECTOR_MAP.get(data.getPluginName());
            Optional.ofNullable(selectorDataList).ifPresent(list -> list.removeIf(e -> e.getId().equals(data.getId())));
        });
    }
    
    public List<SelectorData> obtainSelectorData(final String pluginName) {
        return SELECTOR_MAP.get(pluginName);
    }

    public void cacheRuleData(final RuleData ruleData) {
        Optional.ofNullable(ruleData).ifPresent(this::ruleAccept);
    }
    
    public void removeRuleData(final RuleData ruleData) {
        Optional.ofNullable(ruleData).ifPresent(data -> {
            final List<RuleData> ruleDataList = RULE_MAP.get(data.getSelectorId());
            Optional.ofNullable(ruleDataList).ifPresent(list -> list.removeIf(rule -> rule.getId().equals(data.getId())));
        });
    }

    public List<RuleData> obtainRuleData(final String selectorId) {
        return RULE_MAP.get(selectorId);
    }
}
```

&ensp;&ensp;&ensp;&ensp;从上面的代码可以看出，这是一个静态单例类，从各个方法的名字大致可以看出基本都是对插件配置数据、选择器配置数据、规则配置数据进行的增删改查

&ensp;&ensp;&ensp;&ensp;其中明显的可以看出 cachexxxx , removexxxx 就是对应的更新（增加和修改对map来说一样的操作）和删除操作

&ensp;&ensp;&ensp;&ensp;下面我们来看下， cachexxx、removexxxxx 的方法被哪些地方进行了引用

&ensp;&ensp;&ensp;&ensp;我们使用IDEA，Alt+F7 查看用那些地方进行了使用，通篇查找下来，处理 BaseDataCache类本身和一些测试相关的类，只有CommonPluginDataSubscriber这个类进行了调用，那这个类就是数据同步的核心类了，大致代码如下：

```java
public class CommonPluginDataSubscriber implements PluginDataSubscriber {
    .......

    // 下面这类函数又进行了一次封装，我们可以看到这里有那么一点小瑕疵，命名有点不统一；onSubscribe这个函数取的不是太好，不能见名识意，有点宽泛了
    @Override
    public void onSubscribe(final PluginData pluginData) {
        subscribeDataHandler(pluginData, DataEventTypeEnum.UPDATE);
    }
    
    @Override
    public void unSubscribe(final PluginData pluginData) {
        subscribeDataHandler(pluginData, DataEventTypeEnum.DELETE);
    }

    public void onSelectorSubscribe(final SelectorData selectorData) {
        subscribeDataHandler(selectorData, DataEventTypeEnum.UPDATE);
    }
    
    @Override
    public void unSelectorSubscribe(final SelectorData selectorData) {
        subscribeDataHandler(selectorData, DataEventTypeEnum.DELETE);
    }

    @Override
    public void onRuleSubscribe(final RuleData ruleData) {
        subscribeDataHandler(ruleData, DataEventTypeEnum.UPDATE);
    }
    
    @Override
    public void unRuleSubscribe(final RuleData ruleData) {
        subscribeDataHandler(ruleData, DataEventTypeEnum.DELETE);
    }
    
    private <T> void subscribeDataHandler(final T classData, final DataEventTypeEnum dataType) {
        Optional.ofNullable(classData).ifPresent(data -> {
            if (data instanceof PluginData) {
                PluginData pluginData = (PluginData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    // 插件配置数据的更新
                    BaseDataCache.getInstance().cachePluginData(pluginData);
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.handlerPlugin(pluginData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    // 插件配置数据的删除
                    BaseDataCache.getInstance().removePluginData(pluginData);
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.removePlugin(pluginData));
                }
            } else if (data instanceof SelectorData) {
                SelectorData selectorData = (SelectorData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    // 选择器数据的更新
                    BaseDataCache.getInstance().cacheSelectData(selectorData);
                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.handlerSelector(selectorData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    // 选择器数据的删除
                    BaseDataCache.getInstance().removeSelectData(selectorData);
                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.removeSelector(selectorData));
                }
            } else if (data instanceof RuleData) {
                RuleData ruleData = (RuleData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    // 规则数据的更新
                    BaseDataCache.getInstance().cacheRuleData(ruleData);
                    Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.handlerRule(ruleData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    // 规则数据的删除
                    BaseDataCache.getInstance().removeRuleData(ruleData);
                    Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.removeRule(ruleData));
                }
            }
        });
    }
}
```

&ensp;&ensp;&ensp;&ensp;在上面的代码中，我们看它又对相关数据的更新做了一层封装，在其中我们找到一点命名不是太好的地方

&ensp;&ensp;&ensp;&ensp;下面我们来看下这些：onSelectorSubscribe 之类的函数在什么地方被调用了，通过Alt+F7发现（排查测试类和本身），有下面的三个模块进行引用：

- soul-sync-data-http : PluginDataRefresh
- soul-sync-data-nacos : NacosCacheHandler
- soul-sync-data-websocket : PluginDataHandler
- soul-sync-data-zookeeper : ZookeeperSyncDataService

&ensp;&ensp;&ensp;&ensp;结合前面的了解，可以看到是四大同步方式模块：HTTP、Nacos、websocket、zookeeper

&ensp;&ensp;&ensp;&ensp;大致浏览了一下，这思路同步模块的代码差异比较大，那就是四篇的分析文章了啊

## 总结
&ensp;&ensp;&ensp;&ensp;此篇文章大致梳理了数据同步比较基础的类：

- BaseDataCache : 配置缓存，提供更新、删除、查询
- CommonPluginDataSubscriber : 对BaseDataCache的封装，提供更新和删除
- 同步模块：调用CommonPluginDataSubscriber的接口进行数据同步
    - soul-sync-data-http : PluginDataRefresh
    - soul-sync-data-nacos : NacosCacheHandler
    - soul-sync-data-websocket : PluginDataHandler
    - soul-sync-data-zookeeper : ZookeeperSyncDataService

&ensp;&ensp;&ensp;&ensp;此篇文章先对数据同步有个大致的了解，下面的文章开始进行四大同步模块的分析

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
- [Soul网关源码阅读（九）插件配置加载初探](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB9-%E6%8F%92%E4%BB%B6%E9%85%8D%E7%BD%AE%E5%8A%A0%E8%BD%BD%E5%88%9D%E6%8E%A2.md)
- [Soul网关源码阅读（十）自定义简单插件编写](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB10-%E8%87%AA%E5%AE%9A%E4%B9%89%E7%AE%80%E5%8D%95%E6%8F%92%E4%BB%B6%E7%BC%96%E5%86%99.md)
- [Soul网关源码阅读（十一）请求处理小结](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB11-%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E5%B0%8F%E7%BB%93.md)
- [Soul网关源码阅读（十二）数据同步初探-Bootstrap端](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB12-%E6%95%B0%E6%8D%AE%E5%90%8C%E6%AD%A5%E5%88%9D%E6%8E%A2.md)
- [Soul网关源码阅读（十三）Websocket同步数据-Bootstrap端](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB13-websocket%E5%90%8C%E6%AD%A5%E6%95%B0%E6%8D%AE-Bootstrap%E7%AB%AF.md)
- [Soul网关源码阅读（十四）HTTP数据同步-Bootstrap端](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB14-HTTP%E6%95%B0%E6%8D%AE%E5%90%8C%E6%AD%A5-Bootstrap%E7%AB%AF.md)

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
- [Soul网关源码阅读（九）插件配置加载初探](https://juejin.cn/post/6920074307590684685/)
- [Soul网关源码阅读（十）自定义简单插件编写](https://juejin.cn/post/6920142348617777166)
- [Soul网关源码阅读（十一）请求处理小结](https://juejin.cn/post/6920596034171174925)
- [Soul网关源码阅读（十二）数据同步初探](https://juejin.cn/post/6920596173925384206)
- [Soul网关源码阅读（十三）Websocket同步数据-Bootstrap端](https://juejin.cn/post/6920596028505178125)
- [Soul网关源码阅读（十四）HTTP数据同步-Bootstrap端](https://juejin.cn/post/6920597298674302983)

- [Soul网关源码阅读番外篇（一） HTTP参数请求错误](https://juejin.cn/post/6918947689564471309)