# Soul网关源码解析（十七）Nacos数据同步解析-Bootstrap端
***
## 简介
&ensp;&ensp;&ensp;&ensp;基于上篇：[Soul网关源码阅读（十五）Zookeeper数据同步-Bootstrap端](https://juejin.cn/post/6920764643967238151)，这篇我们要研究下Nacos数据同步原理

## 概览
&ensp;&ensp;&ensp;&ensp;Nacos数据同步和zookeeper数据同步非常相似，都是通过监听数据变化的手段来进行实现的，而且从Debug的数据接收情况来看，形式上表现的也是增量更新的

&ensp;&ensp;&ensp;&ensp;但猫大人推荐的同步方式是websocket和zookeeper，原因是他们能增量更新，并没有提到nacos。在提到HTTP长轮询的时候，提了以后Nacos的实现，则猜测Nacos内部实现的方式是长轮询，所以不推荐Nacos数据同步方式。但这只是猜测，需要后面研究下Nacos才能验证，这里先大胆猜一下

&ensp;&ensp;&ensp;&ensp;通过分析，Nacos的数据同步方式流程基本如下：

- 1.构造Nacos相关服务
- 2.启动监听目前的五种数据类型
- 3.收到改动的时候调用相应的subscribe进行数据更新

&ensp;&ensp;&ensp;&ensp;Nacos的数据更新代码有些令人疑惑，在数据变化收到数据的时候，先进行unSubscribe，再onSubscribe，相当的令人不解

&ensp;&ensp;&ensp;&ensp;而且Nacos好像收不到数据删除通知

&ensp;&ensp;&ensp;&ensp;具体解析过程请看源码Debug环节

## 示例运行
&ensp;&ensp;&ensp;&ensp;2021.1.23日之前的Soul主分支版本Nacos同步存在问题，运行需要参考上篇：[Soul网关源码阅读（十五）Zookeeper数据同步-Bootstrap端](https://juejin.cn/post/6920764643967238151),进行配置，配置运行后，我们进入源码Debug环节

## 源码Debug
### 启动配置跟踪
&ensp;&ensp;&ensp;&ensp;首先寻找切入点，在之前的[Soul网关源码阅读（十二）数据同步初探](https://juejin.cn/post/6920596173925384206)中，我们知道了Nacos数据同步的入口类是：

- soul-sync-data-nacos : NacosCacheHandler

&ensp;&ensp;&ensp;&ensp;我们找到相应的类，其构造函数如下，我们看到了熟悉的Subscribe关键字，通过调用他们实现本地缓存的更新

```java
public class NacosCacheHandler {
    public NacosCacheHandler(final ConfigService configService, final PluginDataSubscriber pluginDataSubscriber,
                             final List<MetaDataSubscriber> metaDataSubscribers,
                             final List<AuthDataSubscriber> authDataSubscribers) {
        this.configService = configService;
        this.pluginDataSubscriber = pluginDataSubscriber;
        this.metaDataSubscribers = metaDataSubscribers;
        this.authDataSubscribers = authDataSubscribers;
    }

    protected void updatePluginMap(final String configInfo) {
        try {
            // Fix bug #656(https://github.com/dromara/soul/issues/656)
            List<PluginData> pluginDataList = new ArrayList<>(GsonUtils.getInstance().toObjectMap(configInfo, PluginData.class).values());
            pluginDataList.forEach(pluginData -> Optional.ofNullable(pluginDataSubscriber).ifPresent(subscriber -> {
                subscriber.unSubscribe(pluginData);
                subscriber.onSubscribe(pluginData);
            }));
        } catch (JsonParseException e) {
            log.error("sync plugin data have error:", e);
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;我们在上面的构造函数上打上断点，看看其调用栈

&ensp;&ensp;&ensp;&ensp;我们来到下面这个类，我们看到了其继承了NacosCacheHandler,构造好相应的数据后，start启动，在其函数中，我们看到了和zookeeper监听非常相似的函数，我们猜测这个函数就是监听的函数。依据前面的经验，在里面应该进行了一些初始化的工作后进行变化监听，我们后面具体调试看看。继续在构造函数上打上断点，再往上看看

```java
public class NacosSyncDataService extends NacosCacheHandler implements AutoCloseable, SyncDataService {

    public NacosSyncDataService(final ConfigService configService, final PluginDataSubscriber pluginDataSubscriber,
                                final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {

        super(configService, pluginDataSubscriber, metaDataSubscribers, authDataSubscribers);
        start();
    }

    public void start() {
        watcherData(PLUGIN_DATA_ID, this::updatePluginMap);
        watcherData(SELECTOR_DATA_ID, this::updateSelectorMap);
        watcherData(RULE_DATA_ID, this::updateRuleMap);
        watcherData(META_DATA_ID, this::updateMetaDataMap);
        watcherData(AUTH_DATA_ID, this::updateAuthMap);
    }

    @Override
    public void close() {
        LISTENERS.forEach((dataId, lss) -> {
            lss.forEach(listener -> getConfigService().removeListener(dataId, GROUP, listener));
            lss.clear();
        });
        LISTENERS.clear();
    }
}
```

&ensp;&ensp;&ensp;&ensp;在上面的构造函数打上断点后，我们跟踪调用栈，来到熟悉的Spring配置，这里配置了Nacos相关的东西，构造nacosSyncDataService的时候启动了Nacos监听

```java
public class NacosSyncDataConfiguration {

    @Bean
    public SyncDataService nacosSyncDataService(final ObjectProvider<ConfigService> configService, final ObjectProvider<PluginDataSubscriber> pluginSubscriber,
                                           final ObjectProvider<List<MetaDataSubscriber>> metaSubscribers, final ObjectProvider<List<AuthDataSubscriber>> authSubscribers) {
        log.info("you use nacos sync soul data.......");
        return new NacosSyncDataService(configService.getIfAvailable(), pluginSubscriber.getIfAvailable(),
                metaSubscribers.getIfAvailable(Collections::emptyList), authSubscribers.getIfAvailable(Collections::emptyList));
    }

    @Bean
    public ConfigService nacosConfigService(final NacosConfig nacosConfig) throws Exception {
        Properties properties = new Properties();
        if (nacosConfig.getAcm() != null && nacosConfig.getAcm().isEnabled()) {
            properties.put(PropertyKeyConst.ENDPOINT, nacosConfig.getAcm().getEndpoint());
            properties.put(PropertyKeyConst.NAMESPACE, nacosConfig.getAcm().getNamespace());
            properties.put(PropertyKeyConst.ACCESS_KEY, nacosConfig.getAcm().getAccessKey());
            properties.put(PropertyKeyConst.SECRET_KEY, nacosConfig.getAcm().getSecretKey());
        } else {
            properties.put(PropertyKeyConst.SERVER_ADDR, nacosConfig.getUrl());
            properties.put(PropertyKeyConst.NAMESPACE, nacosConfig.getNamespace());
        }
        return NacosFactory.createConfigService(properties);
    }

    @Bean
    @ConfigurationProperties(prefix = "soul.sync.nacos")
    public NacosConfig nacosConfig() {
        return new NacosConfig();
    }
}
```

### 数据初始化与监听
&ensp;&ensp;&ensp;&ensp;下面我们回过头来看看，我们取消所有的断点，将断点打在下面start函数中，看看watcherData的具体逻辑

```java
public class NacosSyncDataService extends NacosCacheHandler implements AutoCloseable, SyncDataService {

    public void start() {
        watcherData(PLUGIN_DATA_ID, this::updatePluginMap);
        watcherData(SELECTOR_DATA_ID, this::updateSelectorMap);
        watcherData(RULE_DATA_ID, this::updateRuleMap);
        watcherData(META_DATA_ID, this::updateMetaDataMap);
        watcherData(AUTH_DATA_ID, this::updateAuthMap);
    }
}
```

&ensp;&ensp;&ensp;&ensp;打上断点重启后，我们进入到watcherData内部

&ensp;&ensp;&ensp;&ensp;初看我们可能有点懵，一个函数通用调用就监听了我们之前提到的五种数据变化

&ensp;&ensp;&ensp;&ensp;通过跟踪调试，发现其是靠上面函数中的类型和传入的函数进行区分，从而达到一个函数通用调用监听五种数据的，应该是其内部的实现机制

&ensp;&ensp;&ensp;&ensp;大致是根据传入的数据类型id，调用相应的处理函数，这里知晓其大意即可

```java
public class NacosCacheHandler {

    protected void watcherData(final String dataId, final OnChange oc) {
        Listener listener = new Listener() {
            @Override
            public void receiveConfigInfo(final String configInfo) {
                // 在这里没有发现属性的updatePluginMap之类的
                oc.change(configInfo);
            }

            @Override
            public Executor getExecutor() {
                return null;
            }
        };
        // 这么通过调试，发现触发了初始化操作，从全量的同步一次数据
        oc.change(getConfigAndSignListener(dataId, listener));
        // 这里大致看出是添加进监听列表中之类的
        LISTENERS.getOrDefault(dataId, new ArrayList<>()).add(listener);
    }

    protected interface OnChange {
        void change(String changeData);
    }
}
```

&ensp;&ensp;&ensp;&ensp;我们在下面的更新插件数据的函数中打上断点，发现在进行第一次启动初始化和修改Admin后台管理界面的插件数据后，都会走到下面的函数：

```java
public class NacosCacheHandler {

    protected void updatePluginMap(final String configInfo) {
        try {
            // Fix bug #656(https://github.com/dromara/soul/issues/656)
            List<PluginData> pluginDataList = new ArrayList<>(GsonUtils.getInstance().toObjectMap(configInfo, PluginData.class).values());
            pluginDataList.forEach(pluginData -> Optional.ofNullable(pluginDataSubscriber).ifPresent(subscriber -> {
                // 这里就非常的奇怪，先删除一次数据，再更新一次数据
                subscriber.unSubscribe(pluginData);
                subscriber.onSubscribe(pluginData);
            }));
        } catch (JsonParseException e) {
            log.error("sync plugin data have error:", e);
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;从上面函数大意中，我们大意看出：这是个进行插件信息更新的函数，当收到数据的时候，将其序列化，然后调用相应的subscribe

&ensp;&ensp;&ensp;&ensp;但是我们发现了一个奇怪的逻辑，它先调用删除的逻辑，然后调用更新的逻辑，如前面我们所知，在插件数据进行更新的时候，他们会最终都在走入下面的subscribeDataHandler函数逻辑

&ensp;&ensp;&ensp;&ensp;而进行更新和删除无非就是对本地缓存中的Map进行操作，而在上面的先un再on的操作下，unSubscribe删除数据是不起作用的

&ensp;&ensp;&ensp;&ensp;而且更新数据也不用先删除再添加，直接put替换掉原理的数据即可，不用先删除

&ensp;&ensp;&ensp;&ensp;也就是这个unSubscribe可能是无效和多余的

```java
public class CommonPluginDataSubscriber implements PluginDataSubscriber {
   
    @Override
    public void onSubscribe(final PluginData pluginData) {
        subscribeDataHandler(pluginData, DataEventTypeEnum.UPDATE);
    }
    
    @Override
    public void unSubscribe(final PluginData pluginData) {
        subscribeDataHandler(pluginData, DataEventTypeEnum.DELETE);
    }
    
    private <T> void subscribeDataHandler(final T classData, final DataEventTypeEnum dataType) {
        Optional.ofNullable(classData).ifPresent(data -> {
            if (data instanceof PluginData) {
                PluginData pluginData = (PluginData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    // 最终都会进行Map的更新操作
                    BaseDataCache.getInstance().cachePluginData(pluginData);
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.handlerPlugin(pluginData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    BaseDataCache.getInstance().removePluginData(pluginData);
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.removePlugin(pluginData));
                }
            else {
                ......
            }
        });
    }
}
```

&ensp;&ensp;&ensp;&ensp;而且通过进一步的Debug和分析，发现Nacos似乎不能监听到数据的删除事件，那就是说如果数据失效没有用了，在Bootstrap没有重启的情况下，这些无效的数据会一直占用内存

&ensp;&ensp;&ensp;&ensp;如果是上面分析的那样，那这个Nacos同步机制就有些问题了

## 总结
&ensp;&ensp;&ensp;&ensp;这篇大致分析了Nacos的大致数据同步原理，知道同步流程大致如下：

- NacosSyncDataConfiguration : Nacos启动配置
- NacosSyncDataService
  - 1.初始化读取全量数据进行本地缓存刷新
  - 2.启动数据变化监听
  - 3.接收变化数据，调用相应的subScribe进行相应的更新

&ensp;&ensp;&ensp;&ensp;同时我们发现可能还存在的问题：

- 1.无法接收到数据删除数据
- 2.在数据变化监听处理函数中，unSubscribe函数可能是无用和多余的

