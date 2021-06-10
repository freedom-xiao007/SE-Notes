# Soul网关源码解析（十八）Zookeeper数据同步初探-Admin端
***
## 简介
&ensp;&ensp;&ensp;&ensp;本篇文章来初步探索下Soul-Admin端Zookeeper如何初始化数据、处理更新数据的流程

## 概览
&ensp;&ensp;&ensp;&ensp;从朱明老哥的：[Soul网关源码分析-11期](https://blog.csdn.net/zm469568595/article/details/113065463)文章中定位到Soul-Admin中Zookeeper数据同步相关的配置类位置，而后进行探索分析

&ensp;&ensp;&ensp;&ensp;了解到在没有数据的情况下，会从数据库中读取所有的数据，更新到Zookeeper中；而有数据的情况下，不会进行更新

&ensp;&ensp;&ensp;&ensp;通过在插件更新的处理函数中，通过调用栈，初步梳理得到Zookeeper同步更新数据的大致流程

- 1.后台管理界面（或者注册Client，猜测）发起请求到HTTP接口，触发数据更新
- 2.Admin后台接口触发，调用相应Service进行处理
- 3.事件发布中心进行事件发布
- 4.Zookeeper数据同步组件收到相关事件，进行处理

&ensp;&ensp;&ensp;&ensp;具体解析日志请看源码Debug

## 源码Debug
### 寻找切入点
&ensp;&ensp;&ensp;&ensp;首先根据朱明老哥的：[Soul网关源码分析-11期](https://blog.csdn.net/zm469568595/article/details/113065463)文章定位到Admin模块中的数据同步的核心配置类，代码如下：

```java
@Configuration
public class DataSyncConfiguration {

    @Configuration
    @ConditionalOnProperty(prefix = "soul.sync.zookeeper", name = "url")
    @Import(ZookeeperConfiguration.class)
    static class ZookeeperListener {

        /**
         * Config event listener data changed listener.
         */
        @Bean
        @ConditionalOnMissingBean(ZookeeperDataChangedListener.class)
        public DataChangedListener zookeeperDataChangedListener(final ZkClient zkClient) {
            return new ZookeeperDataChangedListener(zkClient);
        }

        /**
         * Zookeeper data init zookeeper data init.
         */
        @Bean
        @ConditionalOnMissingBean(ZookeeperDataInit.class)
        public ZookeeperDataInit zookeeperDataInit(final ZkClient zkClient, final SyncDataService syncDataService) {
            return new ZookeeperDataInit(zkClient, syncDataService);
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;在上面我们看到Websocket、HTTP、Zookeeper、Nacos数据同步的相关配置（这里只列出Zookeeper），我们可以看到一个是监听的，一个是初始化的

&ensp;&ensp;&ensp;&ensp;这里有个小插曲，在Websocket的配置注解如下：

```java
@Configuration
public class DataSyncConfiguration {

    @Configuration
    @ConditionalOnProperty(name = "soul.sync.websocket.enabled", havingValue = "true", matchIfMissing = true)
    @EnableConfigurationProperties(WebsocketSyncProperties.class)
    static class WebsocketListener {
        .......
    }
}
```

&ensp;&ensp;&ensp;&ensp;在前面的文章中，我们遇到一个将Websocket同步配置给注释掉，但发现Admin还是开启了Websocket的同步，其中的原因就在这，从上面的注解中可以看到，如果没有是否开启的值，那默认就为true（表示开启）

&ensp;&ensp;&ensp;&ensp;那Admin关闭Websocket同步就只有两种方式：

- 1.将配置enabled设置为false
- 2.将Websocket starter依赖去掉（Admin模块如果有的话）


### 数据初始化
&ensp;&ensp;&ensp;&ensp;下面我们来探索下Zookeeper数据的初始化，下面的类可以看出是一个初始化的类，并且是启动时运行的

```java
public class ZookeeperDataInit implements CommandLineRunner {

    @Override
    public void run(final String... args) {
        String pluginPath = ZkPathConstants.PLUGIN_PARENT;
        String authPath = ZkPathConstants.APP_AUTH_PARENT;
        String metaDataPath = ZkPathConstants.META_DATA;
        if (!zkClient.exists(pluginPath) && !zkClient.exists(authPath) && !zkClient.exists(metaDataPath)) {
            // 从名称可以看出是同步所有的数据
            syncDataService.syncAll(DataEventTypeEnum.REFRESH);
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;我们在run函数上打上断点，果然在重启Admin后进入该断点，但发现在Debug的时候没有能进入syncAll函数

&ensp;&ensp;&ensp;&ensp;猜测是Zookeeper为空才进行初始化

&ensp;&ensp;&ensp;&ensp;我们将Zookeeper中的soul节点删除，清空所有的数据

&ensp;&ensp;&ensp;&ensp;重启Admin进入断点，并进入run函数，发现成功进入syncAll函数，我们来看看其代码：

```java
@Service("syncDataService")
public class SyncDataServiceImpl implements SyncDataService {

    // 基本逻辑就是从数据库中读取数据，然后发布事件
    // 这里读取了目前的五种数据并进行事件发布
    @Override
    public boolean syncAll(final DataEventTypeEnum type) {
        // type : refresh
        // 里面也是读取数据数据，然后发布事件
        appAuthService.syncData();
        List<PluginData> pluginDataList = pluginService.listAll();
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.PLUGIN, type, pluginDataList));
        List<SelectorData> selectorDataList = selectorService.listAll();
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.SELECTOR, type, selectorDataList));
        List<RuleData> ruleDataList = ruleService.listAll();
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.RULE, type, ruleDataList));
        metaDataService.syncData();
        return true;
    }
}
```

&ensp;&ensp;&ensp;&ensp;在上面的函数中，pluginService是我们熟悉不过的CRUD相关的东西了，这里就不多说了

&ensp;&ensp;&ensp;&ensp;可以明显的看出，初始化就是从数据库中读取所有的数据，然后发布数据更新事件，细节这里先不看，我们下面来看看一个数据更新的处理流程


### 插件数据更新处理流程
&ensp;&ensp;&ensp;&ensp;我们看看另外一个数据监听的类：ZookeeperDataChangedListener,在其中找到了非常可疑的插件更新的处理函数，我们在其上打上断点

&ensp;&ensp;&ensp;&ensp;然后我们在Admin后台管理界面中将限流插件的状态进行修改，果然进入了断点之中，稍微跟踪了下，最终的数据会写入Zookeeper节点中。这样Zookeeper数据发生变化，就会推送数据给Bootstrap，Bootstrap的数据也就跟着更新了

```java
public class ZookeeperDataChangedListener implements DataChangedListener {

    @Override
    public void onPluginChanged(final List<PluginData> changed, final DataEventTypeEnum eventType) {
        for (PluginData data : changed) {
            final String pluginPath = ZkPathConstants.buildPluginPath(data.getName());
            // delete
            if (eventType == DataEventTypeEnum.DELETE) {
                deleteZkPathRecursive(pluginPath);
                final String selectorParentPath = ZkPathConstants.buildSelectorParentPath(data.getName());
                deleteZkPathRecursive(selectorParentPath);
                final String ruleParentPath = ZkPathConstants.buildRuleParentPath(data.getName());
                deleteZkPathRecursive(ruleParentPath);
                continue;
            }
            //create or update
            upsertZkNode(pluginPath, data);
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;我们接着看调用栈，看看其处理流程

&ensp;&ensp;&ensp;&ensp;跟踪调用栈，来到下面的函数，可以看到和Bootstrap的挺像的，下面接收事件后，根据其GoupKey，进行分发处理，getSource应该就是具体的插件数据，另外一个是事件类型，是更新还是删除

```java
public class DataChangedEventDispatcher implements ApplicationListener<DataChangedEvent>, InitializingBean {

    @Override
    @SuppressWarnings("unchecked")
    public void onApplicationEvent(final DataChangedEvent event) {
        for (DataChangedListener listener : listeners) {
            switch (event.getGroupKey()) {
                case APP_AUTH:
                    listener.onAppAuthChanged((List<AppAuthData>) event.getSource(), event.getEventType());
                    break;
                case PLUGIN:
                    listener.onPluginChanged((List<PluginData>) event.getSource(), event.getEventType());
                    break;
                case RULE:
                    listener.onRuleChanged((List<RuleData>) event.getSource(), event.getEventType());
                    break;
                case SELECTOR:
                    listener.onSelectorChanged((List<SelectorData>) event.getSource(), event.getEventType());
                    break;
                case META_DATA:
                    listener.onMetaDataChanged((List<MetaData>) event.getSource(), event.getEventType());
                    break;
                default:
                    throw new IllegalStateException("Unexpected value: " + event.getGroupKey());
            }
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;我们继续跟踪调用栈来到下面的函数，发现是一个CRUD的Service，其中的处理逻辑是：先进行数据库更新，然后发布事件

```java
@Service("pluginService")
public class PluginServiceImpl implements PluginService {

    @Override
    @Transactional(rollbackFor = Exception.class)
    public String createOrUpdate(final PluginDTO pluginDTO) {
        final String msg = checkData(pluginDTO);
        if (StringUtils.isNoneBlank(msg)) {
            return msg;
        }
        PluginDO pluginDO = PluginDO.buildPluginDO(pluginDTO);
        DataEventTypeEnum eventType = DataEventTypeEnum.CREATE;
        if (StringUtils.isBlank(pluginDTO.getId())) {
            pluginMapper.insertSelective(pluginDO);
        } else {
            eventType = DataEventTypeEnum.UPDATE;
            pluginMapper.updateSelective(pluginDO);
        }

        // publish change event.
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.PLUGIN, eventType,
                Collections.singletonList(PluginTransfer.INSTANCE.mapToData(pluginDO))));
        return StringUtils.EMPTY;
    }
}
```

&ensp;&ensp;&ensp;&ensp;继续跟，我们来到Controller,其逻辑也很简单，调用相应的Service即可，这里也不过多赘述了。到这里大致的流程就走了一遍

```java
@RestController
@RequestMapping("/plugin")
public class PluginController {
    @PutMapping("/{id}")
    public SoulAdminResult updatePlugin(@PathVariable("id") final String id, @RequestBody final PluginDTO pluginDTO) {
        Objects.requireNonNull(pluginDTO);
        pluginDTO.setId(id);
        final String result = pluginService.createOrUpdate(pluginDTO);
        if (StringUtils.isNoneBlank(result)) {
            return SoulAdminResult.error(result);
        }
        return SoulAdminResult.success(SoulResultMessage.UPDATE_SUCCESS);
    }
}
```

## 总结
&ensp;&ensp;&ensp;&ensp;本篇文章进行了初步探索的Admin端的Zookeeper数据同步的处理流程，大致可以分为初始化和数据更新（包括删除）的处理流程

- 数据初始化：ZookeeperDataInit
  - Zookeeper中不为空，则不进行初始化操作
  - Zookeeper为空，则进行初始化操作，从数据库中读取数据，放入到其中

- 数据处理流程(监听类ZookeeperDataChangedListener)
  - HTTP接口调用：可以是管理后台；也可以是服务注册Client
  - Service调用：更新数据库中的数据，调用发布事件接口
  - 发布事件：发布事件到数据同步监听中
  - 数据更新：接收到事件后，进行更新（Websocket进行推送、Zookeeper写入、HTTP更新MD5、Nacos写入）

&ensp;&ensp;&ensp;&ensp;根据此次的分析，大致猜测如上

&ensp;&ensp;&ensp;&ensp;此次还意外解决之前的Websocket数据同步关闭失效问题