# Soul网关源码解析（十九）Nacos数据同步初始化修复-Admin端
***
## 简介
&ensp;&ensp;&ensp;&ensp;本篇文章记录一次Nacos数据同步解析中发现问题，并提交PR，成功合并，解决Soul-Admin的Nacos数据同步初始化问题

## 代码版本（Version）
```bash
commit b60fb246977ff36fa58ed0a7a9ce22d57dd30323 (upstream/master)
Author: cottom <11937539+cottom@users.noreply.github.com>
Date:   Sun Jan 24 22:06:34 2021 +0800
```

## 问题简述（Problem Description）
&ensp;&ensp;&ensp;&ensp;1.在配置后Nacos，并在Nacos中新建对应namespace后，启动Admin，发现在Nacos中并无数据，需要在Admin管理后台页面中收到点击同步数据，数据才会同步到Nacos中

&ensp;&ensp;&ensp;&ensp;2.在上一步同步中，rule相应的Id会建立在Nacos中，但其中并没有数据，导致访问失败

## 问题详情及解决方案探索
### 1.Nacos启动无法初始化问题
#### 相关代码
&ensp;&ensp;&ensp;&ensp;在同步配置类中，Nacos的配置只是简单启动了Nacos监听，没有类似Zookeeper的还有一个初始化的类，代码大致如下：

```java
@Configuration
public class DataSyncConfiguration {

    // 这里只启动了一个监听的，并且在NacosDataChangedListener中，并没有发现任何的初始化操作
    @Configuration
    @ConditionalOnProperty(prefix = "soul.sync.nacos", name = "url")
    @Import(NacosConfiguration.class)
    static class NacosListener {

        @Bean
        @ConditionalOnMissingBean(NacosDataChangedListener.class)
        public DataChangedListener nacosDataChangedListener(final ConfigService configService) {
            return new NacosDataChangedListener(configService);
        }
    }
}
```

#### 解决方案探索
&ensp;&ensp;&ensp;&ensp;可以解决Zookeeper的代码，新建一个初始化数据的类，判断其数据节点是否存在，如果不存在，则进行从数据库中读取所有数据，发布事件，触发Nacos的初始化

```java
public class DataSyncConfiguration {
    @Configuration
    @ConditionalOnProperty(prefix = "soul.sync.zookeeper", name = "url")
    @Import(ZookeeperConfiguration.class)
    static class ZookeeperListener {

        @Bean
        @ConditionalOnMissingBean(ZookeeperDataChangedListener.class)
        public DataChangedListener zookeeperDataChangedListener(final ZkClient zkClient) {
            return new ZookeeperDataChangedListener(zkClient);
        }

        @Bean
        @ConditionalOnMissingBean(ZookeeperDataInit.class)
        public ZookeeperDataInit zookeeperDataInit(final ZkClient zkClient, final SyncDataService syncDataService) {
            return new ZookeeperDataInit(zkClient, syncDataService);
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;初步写了一个Nacos初始化触发的类，模仿的ZookeeperDataInit,在run函数中，首先判断插件、元数据、认证数据的Id数据是否存在，如果不存在，则想Zookeeper一样触发同步所有数据的事件

```java
public class NacosDataInit implements CommandLineRunner {

    private static final String GROUP = "DEFAULT_GROUP";

    private final ConfigService configService;
    private final SyncDataService syncDataService;

    public NacosDataInit(final ConfigService configService, final SyncDataService syncDataService) {
        this.configService = configService;
        this.syncDataService = syncDataService;
    }

    @Override
    public void run(String... args) throws Exception {
        String pluginId = NacosDataIdConstants.PLUGIN_DATA_ID;
        String authId = NacosDataIdConstants.AUTH_DATA_ID;
        String metaDataId = NacosDataIdConstants.META_DATA_ID;
        if (!isExists(pluginId) && !isExists(authId) && !isExists(metaDataId)) {
            syncDataService.syncAll(DataEventTypeEnum.REFRESH);
        }
    }

    @SneakyThrows
    private boolean isExists(String id) {
        return configService.getConfig(id, GROUP, 3000) != null;
    }
}
```

&ensp;&ensp;&ensp;&ensp;经过初步测试，上面的初始化类，在Nacos为空时，重启Admin，能初步同步数据到Nacos中

&ensp;&ensp;&ensp;&ensp;但还是存在下面的第二个问题：rule规则为空

### 2.同步插件数据后，rule规则没有进行同步，仍为空
#### 相关代码
&ensp;&ensp;&ensp;&ensp;如下代码所示，在进行插件同步（触发条件为在Admin管理后台界面点击同步自定义xxx插件），根据代码大意：会进行触发插件数据的更新和其选择器和规则，但调试发现，并不能成功更新规则数据，原因是下一段代码：

```java
public class SyncDataServiceImpl implements SyncDataService {

    public boolean syncPluginData(final String pluginId) {
        PluginVO pluginVO = pluginService.findById(pluginId);
        // 同步插件
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.PLUGIN, DataEventTypeEnum.UPDATE,
                Collections.singletonList(PluginTransfer.INSTANCE.mapDataTOVO(pluginVO))));
        List<SelectorData> selectorDataList = selectorService.findByPluginId(pluginId);
        // 同步选择器
        if (CollectionUtils.isNotEmpty(selectorDataList)) {
            eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.SELECTOR, DataEventTypeEnum.REFRESH, selectorDataList));
            List<RuleData> allRuleDataList = new ArrayList<>();
            for (SelectorData selectData : selectorDataList) {
                List<RuleData> ruleDataList = ruleService.findBySelectorId(selectData.getId());
                allRuleDataList.addAll(ruleDataList);
            }
            // 同步规则
            eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.RULE, DataEventTypeEnum.REFRESH, allRuleDataList));
        }
        return true;
    }
}
```

&ensp;&ensp;&ensp;&ensp;从上面的函数事件发布后，进行规则的更新，来到下面的Nacos规则数据更新代码

&ensp;&ensp;&ensp;&ensp;事件为REFRESH，在其处理逻辑中，我们发现其并能添加进去新的规则：因为ls的取值逻辑会一直为空，本身RULE_MAP就是为空的，取不到值

```java
public void onRuleChanged(final List<RuleData> changed, final DataEventTypeEnum eventType) {
        updateRuleMap(getConfig(RULE_DATA_ID));
        switch (eventType) {
                .......
            case REFRESH:
            // 在Zookeeper的初始化触发MYSELF，那Nacos应该也是一样
            // 在初始化的时候，下面的ls中始终为空，rule无法添加进去
            case MYSELF:
                Set<String> set = new HashSet<>(RULE_MAP.keySet());
                changed.forEach(rule -> {
                    set.remove(rule.getSelectorId());
                    // 始终为空
                    List<RuleData> ls = RULE_MAP
                            .getOrDefault(rule.getSelectorId(), new ArrayList<>())
                            .stream()
                            .sorted(RULE_DATA_COMPARATOR)
                            .collect(Collectors.toList());
                    // 可以直接加进去
                    ls.add(rule);
                    RULE_MAP.put(rule.getSelectorId(), ls);
                });
                RULE_MAP.keySet().removeAll(set);
                break;
            default:
                ......
                break;
        }

        publishConfig(RULE_DATA_ID, RULE_MAP);
    }
```

#### 解决方案探索
&ensp;&ensp;&ensp;&ensp;在REFRESH的逻辑处理中出现了错误，REFRESH的处理逻辑参照zookeeper应该是：

- 1.清空本地的缓存数据
- 2.添加传入的新的数据

&ensp;&ensp;&ensp;&ensp;但在下面原理的逻辑代码中，逻辑不是很清晰，将起来也比较费劲，我们直接修改为下面的代码：

```java
public void onRuleChanged(final List<RuleData> changed, final DataEventTypeEnum eventType) {
        updateRuleMap(getConfig(RULE_DATA_ID));
        switch (eventType) {
                .......
            case REFRESH:
            case MYSELF:
                // 清空本地缓存数据
                SELECTOR_MAP.keySet().removeAll(SELECTOR_MAP.keySet());
                changed.forEach(rule -> {
                    // 根据当前rule对应的selectorid，得到rule所在的列表
                    // 如果为空，则新建一个列表
                    // 如果之前有，那取出来
                    List<RuleData> ls = RULE_MAP
                            .getOrDefault(rule.getSelectorId(), new ArrayList<>())
                            .stream()
                            .sorted(RULE_DATA_COMPARATOR)
                            .collect(Collectors.toList());
                    // 将新的规则添加到selector的列表中
                    ls.add(rule);
                    RULE_MAP.put(rule.getSelectorId(), ls);
                });
                break;
            default:
                ......
                break;
        }

        publishConfig(RULE_DATA_ID, RULE_MAP);
    }
```

&ensp;&ensp;&ensp;&ensp;核心逻辑就是先清空缓存；然后逐个将rule添加到对应的selector的列表中

&ensp;&ensp;&ensp;&ensp;经过初步测试，能解决初始化的时候rule没有数据和同步自定义插件没有更新rule的问题，并且请求访问正常

## 总结
&ensp;&ensp;&ensp;&ensp;在前面的分析中，我们就遇到了Nacos的问题，然后我们先解析了Admin的Zookeeper的同步数据逻辑，以此来对应上我们这次的Nacos同步逻辑。并顺利的找到其相应的bug，并成功修改提交PR，成功合并！

&ensp;&ensp;&ensp;&ensp;运用所学，解决了一个问题，真滴开心！

## 参考链接
- [Nacos Java SDK](https://nacos.io/zh-cn/docs/sdk.html)

## Soul网关源码分析文章列表
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
- [Soul网关源码阅读（十五）Zookeeper数据同步-Bootstrap端](https://juejin.cn/post/6920764643967238151)
- [Soul网关源码阅读（十六）Nacos数据同步示例运行](https://juejin.cn/post/6921170233868845064)
- [Soul网关源码阅读（十七）Nacos数据同步解析-Bootstrap端](https://juejin.cn/post/6921325882753695757/)
- [Soul网关源码阅读（十八）Zookeeper数据同步初探-Admin端](https://juejin.cn/post/6921495273122463751/)