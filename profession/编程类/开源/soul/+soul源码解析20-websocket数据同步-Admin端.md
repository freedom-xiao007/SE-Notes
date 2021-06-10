# Soul网关源码解析（二十）Websocket数据同步-Admin端
***
## 简介
&ensp;&ensp;&ensp;&ensp;本篇文章探索下Soul网关Admin的Websocket数据同步流程

## 概览
&ensp;&ensp;&ensp;&ensp;首先使用Websocket同步方式启动下示例

&ensp;&ensp;&ensp;&ensp;根据前面Zookeeper和Nacos数据同步分析的经验，找到Websocket的事件监听处理的类，在其上打上断点，调试查看初始化流程

&ensp;&ensp;&ensp;&ensp;然后在Admin后台修改插件状态，调试查看数据变更处理流程

&ensp;&ensp;&ensp;&ensp;发现初始化都是从熟悉的syncAll开始，而事件变更处理都是从Controllers入口开始的，具体详情记录情况在源码Debug环节

## 示例运行
&ensp;&ensp;&ensp;&ensp;启动数据库：

```shell script
docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:latest
```

&ensp;&ensp;&ensp;&ensp;首先配置运行Soul-Admin，设置数据同步方式为Websocket

```xml
soul:
  sync:
    websocket:
      enabled: true
```

&ensp;&ensp;&ensp;&ensp;配置Soul-Bootstrap，配置websocket同步方式，运行Soul-Bootstrap，大致如下：

```xml
soul :
    # 把 websocket 数据同步打开
    sync:
        websocket :
             urls: ws://localhost:9095/websocket
```

&ensp;&ensp;&ensp;&ensp;运行Soul-Example-HTTP，注册一些数据用于Debug测试

## 源码Debug
### 初始化流程
&ensp;&ensp;&ensp;&ensp;首先根据前面的经验在Soul-Admin模块，listener.websocket 目录包下找到相应的Websocket事件监听处理类：WebsocketDataChangedListener

&ensp;&ensp;&ensp;&ensp;我们找到插件变更处理的函数，在其上打上端口，重启Admin

&ensp;&ensp;&ensp;&ensp;成功进入断点，我们可以看到下面的函数大意是封装了Websocket格式的数据，然后用Websocket发送出去，细节后面再看，我们先看看调用栈

```java
public class WebsocketDataChangedListener implements DataChangedListener {

    @Override
    public void onPluginChanged(final List<PluginData> pluginDataList, final DataEventTypeEnum eventType) {
        WebsocketData<PluginData> websocketData =
                new WebsocketData<>(ConfigGroupEnum.PLUGIN.name(), eventType.name(), pluginDataList);
        WebsocketCollector.send(GsonUtils.getInstance().toJson(websocketData), eventType);
    }
}
```

&ensp;&ensp;&ensp;&ensp;PS：这个有个小细节，如果没有任何一台Bootstrap或者事件发送，那这个断点不会进入

&ensp;&ensp;&ensp;&ensp;我们跟踪调用栈来到前面文章中熟悉的事件处理分发，这里继续跟下去

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
                // Plugin 触发
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

&ensp;&ensp;&ensp;&ensp;来到又是非常熟悉的全量数据同步函数：从数据库中读取所有的数据，然后发布事件，进行同步

```java
public class SyncDataServiceImpl implements SyncDataService {

    @Override
    public boolean syncAll(final DataEventTypeEnum type) {
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

&ensp;&ensp;&ensp;&ensp;再跟，来到了Websocket相关的，下面这个函数如果是写过Websocket的，那一定非常熟悉了，就是接收到消息，然后调用处理逻辑

&ensp;&ensp;&ensp;&ensp;在调试中，我们看到message的值是MYSELF，猜测Websocket的初始化通信约定应该是收到MYSELF，则同步全量数据给Bootstrap

```java
public class WebsocketCollector {

    @OnMessage
    public void onMessage(final String message, final Session session) {
        // message == MYSELF
        if (message.equals(DataEventTypeEnum.MYSELF.name())) {
            try {
                // 将客户端Session信息进行保存
                ThreadLocalUtil.put(SESSION_KEY, session);
                SpringBeanUtils.getInstance().getBean(SyncDataService.class).syncAll(DataEventTypeEnum.MYSELF);
            } finally {
                ThreadLocalUtil.clear();
            }
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;我们回过头去看看Websocket的发送函数，有两个点需要注意一下：

&ensp;&ensp;&ensp;&ensp;1.是当为MYSELF时，消息只发送给一个特定的客户端

&ensp;&ensp;&ensp;&ensp;在上面的函数中，使用ThreadLocal进行了session的保存，然后取出来，然后利用session进行消息发送

&ensp;&ensp;&ensp;&ensp;这种应该是针对新建立连接的Bootstrap的，发送全量的数据给它

&ensp;&ensp;&ensp;&ensp;2.不是MYSELF，则发送消息给所有的客户端

&ensp;&ensp;&ensp;&ensp;这个应该是事件变更，然后同步数据给所有的客户端

```java
public class WebsocketCollector {

    public static void send(final String message, final DataEventTypeEnum type) {
        if (StringUtils.isNotBlank(message)) {
            if (DataEventTypeEnum.MYSELF == type) {
                Session session = (Session) ThreadLocalUtil.get(SESSION_KEY);
                if (session != null) {
                    sendMessageBySession(session, message);
                }
            } else {
                SESSION_SET.forEach(session -> sendMessageBySession(session, message));
            }
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;在这里看到使用ThreadLocal进行标识获取，但SESSION_KEY都是一样的，它是原理好像自己还有点迷糊，后面再研究下

&ensp;&ensp;&ensp;&ensp;以前我们使用时候，都是在OnMessage中直接进行这种针对特定客户端的处理；或者客户端连接的时候自带ID

&ensp;&ensp;&ensp;&ensp;看了这个，感觉又学到了一手

### 数据变更
&ensp;&ensp;&ensp;&ensp;数据同步走完了，我们在Admin后台管理界面，修改限流插件的状态，然后触发进入了最开始我们打上断点的函数：

```java
public class WebsocketDataChangedListener implements DataChangedListener {

    @Override
    public void onPluginChanged(final List<PluginData> pluginDataList, final DataEventTypeEnum eventType) {
        WebsocketData<PluginData> websocketData =
                new WebsocketData<>(ConfigGroupEnum.PLUGIN.name(), eventType.name(), pluginDataList);
        WebsocketCollector.send(GsonUtils.getInstance().toJson(websocketData), eventType);
    }
}
```

&ensp;&ensp;&ensp;&ensp;跟踪调用栈事件，事件分发跳过，又来到熟悉的：PluginServiceImpl，这里面进行插件数据的更新，然后发布事件

```java
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

&ensp;&ensp;&ensp;&ensp;继续跟踪来到属性的Controllers接口，从这里触发事件更新

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
&ensp;&ensp;&ensp;&ensp;本篇文章进行了初步探索的Admin端的Websocket数据同步的处理流程，大致可以分为初始化和数据更新（包括删除）的处理流程

- 数据初始化：Boostrap发送MYSELF消息，触发所有数据同步

- 数据处理流程(监听类ZookeeperDataChangedListener)
  - HTTP接口调用：可以是管理后台；也可以是服务注册Client
  - Service调用：更新数据库中的数据，调用发布事件接口
  - 发布事件：发布事件到数据同步监听中
  - 数据更新：接收到事件后，进行更新（Websocket进行推送、Zookeeper写入、HTTP更新MD5、Nacos写入）

&ensp;&ensp;&ensp;&ensp;这三篇：Zookeeper、Nacos、Websocket，发现处理流程都是基本相似的，前面的初始化入口、事件触发和分发都是同一个，具体的发送逻辑不一样而已

&ensp;&ensp;&ensp;&ensp;可以看出代码的结构是非常清晰的