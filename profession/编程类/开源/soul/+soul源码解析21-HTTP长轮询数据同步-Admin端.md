# Soul网关源码解析（二十一）HTTP长轮询数据同步-Admin端
***
## 简介
&ensp;&ensp;&ensp;&ensp;本篇文章来探索Soul网关Admin中的HTTP长轮询数据同步流程

## 概览
&ensp;&ensp;&ensp;&ensp;运行示例参考：[Soul网关源码解析（十四）HTTP数据同步-Bootstrap端](https://juejin.cn/post/6920597298674302983)

&ensp;&ensp;&ensp;&ensp;文章分析切入点为上面第十四篇中分析得到的两个请求接口：

- /configs/listener : 数据变化监听接口
- /configs/fetch : 数据获取接口

&ensp;&ensp;&ensp;&ensp;首先对获取接口进行分析，得到其直接从本地缓存中得到数据后返回

&ensp;&ensp;&ensp;&ensp;再来分析数据变化监听端口，见识了一种拉实现推的新操作，请求会在Admin进行等待，超时或者数据有变化后才会返回，非常有意思

&ensp;&ensp;&ensp;&ensp;最后想找出事件发布到更新的流程，踩了点坑，不是很顺利，但感受到了HTTP长轮询数据同步和Websocket、Zookeeper、Nacos有些区别，实现更加复杂一些

## 源码Debug
### 寻找切入点
&ensp;&ensp;&ensp;&ensp;找到第十四篇中HTTP同步需要使用的两个接口位置，代码大致如下：

```java
@RequestMapping("/configs")
public class ConfigController {

    @Resource
    private HttpLongPollingDataChangedListener longPollingListener;

    // 数据获取
    @GetMapping("/fetch")
    public SoulAdminResult fetchConfigs(@NotNull final String[] groupKeys) {
        Map<String, ConfigData<?>> result = Maps.newHashMap();
        for (String groupKey : groupKeys) {
            ConfigData<?> data = longPollingListener.fetchConfig(ConfigGroupEnum.valueOf(groupKey));
            result.put(groupKey, data);
        }
        return SoulAdminResult.success(SoulResultMessage.SUCCESS, result);
    }

    // 数据变化监听
    @PostMapping(value = "/listener")
    public void listener(final HttpServletRequest request, final HttpServletResponse response) {
        longPollingListener.doLongPolling(request, response);
    }
}
```

### 数据获取解析：/configs/fetch
&ensp;&ensp;&ensp;&ensp;首先来看下数据获取接口，我们看到其直接获取各个类型的data后返回

```java
@RequestMapping("/configs")
public class ConfigController {

    @GetMapping("/fetch")
    public SoulAdminResult fetchConfigs(@NotNull final String[] groupKeys) {
        Map<String, ConfigData<?>> result = Maps.newHashMap();
        for (String groupKey : groupKeys) {
            ConfigData<?> data = longPollingListener.fetchConfig(ConfigGroupEnum.valueOf(groupKey));
            result.put(groupKey, data);
        }
        return SoulAdminResult.success(SoulResultMessage.SUCCESS, result);
    }
}
```

&ensp;&ensp;&ensp;&ensp;我们接着来看下 fetchConfig这个函数，看到是根据类型从本地Cache中获取数据返回

```java
public abstract class AbstractDataChangedListener implements DataChangedListener, InitializingBean {
    protected static final ConcurrentMap<String, ConfigDataCache> CACHE = new ConcurrentHashMap<>();

    public ConfigData<?> fetchConfig(final ConfigGroupEnum groupKey) {
        ConfigDataCache config = CACHE.get(groupKey.name());
        switch (groupKey) {
            case APP_AUTH:
                List<AppAuthData> appAuthList = GsonUtils.getGson().fromJson(config.getJson(), new TypeToken<List<AppAuthData>>() {
                }.getType());
                return new ConfigData<>(config.getMd5(), config.getLastModifyTime(), appAuthList);
            case PLUGIN:
                List<PluginData> pluginList = GsonUtils.getGson().fromJson(config.getJson(), new TypeToken<List<PluginData>>() {
                }.getType());
                return new ConfigData<>(config.getMd5(), config.getLastModifyTime(), pluginList);
            case RULE:
                List<RuleData> ruleList = GsonUtils.getGson().fromJson(config.getJson(), new TypeToken<List<RuleData>>() {
                }.getType());
                return new ConfigData<>(config.getMd5(), config.getLastModifyTime(), ruleList);
            case SELECTOR:
                List<SelectorData> selectorList = GsonUtils.getGson().fromJson(config.getJson(), new TypeToken<List<SelectorData>>() {
                }.getType());
                return new ConfigData<>(config.getMd5(), config.getLastModifyTime(), selectorList);
            case META_DATA:
                List<MetaData> metaList = GsonUtils.getGson().fromJson(config.getJson(), new TypeToken<List<MetaData>>() {
                }.getType());
                return new ConfigData<>(config.getMd5(), config.getLastModifyTime(), metaList);
            default:
                throw new IllegalStateException("Unexpected groupKey: " + groupKey);
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;我们去看一下Cache，发现更新函数只能更新 md5 和 时间，json是初始化传入的。这里暂时放过

```java
public class ConfigDataCache {
    
    protected synchronized void update(final String md5, final long lastModifyTime) {
        this.md5 = md5;
        this.lastModifyTime = lastModifyTime;
    }
}
```

&ensp;&ensp;&ensp;&ensp;通过上面的分析，我们知道了fetch接口是直接返回全量时间的，也就是只要调用这个接口，五种类型的数据都会发送过去，数据量还挺大的

### /configs/listener
&ensp;&ensp;&ensp;&ensp;接下来看看监听接口，一看就比较奇怪，没有返回值

```java
@RequestMapping("/configs")
public class ConfigController {

    @PostMapping(value = "/listener")
    public void listener(final HttpServletRequest request, final HttpServletResponse response) {
        longPollingListener.doLongPolling(request, response);
    }
}
```

&ensp;&ensp;&ensp;&ensp;抱着好奇心继续看doLongPolling函数

&ensp;&ensp;&ensp;&ensp;从下面的代码可以看出，它显示看看各个数据类型是否有数据变化。如果有，则生成响应后直接返回（神奇的操作，response直接自己调用返回）；

&ensp;&ensp;&ensp;&ensp;如果没有发送变化，会开启一个线程进行等待，具体内容下面在看

```java
public class HttpLongPollingDataChangedListener extends AbstractDataChangedListener {

    public void doLongPolling(final HttpServletRequest request, final HttpServletResponse response) {

        // compare group md5
        List<ConfigGroupEnum> changedGroup = compareChangedGroup(request);
        String clientIp = getRemoteIp(request);

        // response immediately.
        if (CollectionUtils.isNotEmpty(changedGroup)) {
            this.generateResponse(response, changedGroup);
            log.info("send response with the changed group, ip={}, group={}", clientIp, changedGroup);
            return;
        }

        // listen for configuration changed.
        final AsyncContext asyncContext = request.startAsync();

        // AsyncContext.settimeout() does not timeout properly, so you have to control it yourself
        asyncContext.setTimeout(0L);

        // block client's thread.
        scheduler.execute(new LongPollingClient(asyncContext, clientIp, HttpConstants.SERVER_MAX_HOLD_TIMEOUT));
    }

    // Bootstrap端传过来的应该是它最新的MD5，如果其中有一个类型MD5是新的，那就好添加到group中，如果group不为空，则直接返回
    private List<ConfigGroupEnum> compareChangedGroup(final HttpServletRequest request) {
        List<ConfigGroupEnum> changedGroup = new ArrayList<>(ConfigGroupEnum.values().length);
        for (ConfigGroupEnum group : ConfigGroupEnum.values()) {
            // md5,lastModifyTime
            String[] params = StringUtils.split(request.getParameter(group.name()), ',');
            if (params == null || params.length != 2) {
                throw new SoulException("group param invalid:" + request.getParameter(group.name()));
            }
            String clientMd5 = params[0];
            long clientModifyTime = NumberUtils.toLong(params[1]);
            ConfigDataCache serverCache = CACHE.get(group.name());
            // do check.
            if (this.checkCacheDelayAndUpdate(serverCache, clientMd5, clientModifyTime)) {
                changedGroup.add(group);
            }
        }
        return changedGroup;
    }

    private void generateResponse(final HttpServletResponse response, final List<ConfigGroupEnum> changedGroups) {
        try {
            response.setHeader("Pragma", "no-cache");
            response.setDateHeader("Expires", 0);
            response.setHeader("Cache-Control", "no-cache,no-store");
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.setStatus(HttpServletResponse.SC_OK);
            // 学到了新的技能，response直接自己能调用
            response.getWriter().println(GsonUtils.getInstance().toJson(SoulAdminResult.success(SoulResultMessage.SUCCESS, changedGroups)));
        } catch (IOException ex) {
            log.error("Sending response failed.", ex);
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;我们来看下那个线程任务

&ensp;&ensp;&ensp;&ensp;看着好像是启动了一个定时的任务，时间到了就返回数据，这里细节不太清楚，先放过

```java
public class HttpLongPollingDataChangedListener extends AbstractDataChangedListener {

    class LongPollingClient implements Runnable {
        @Override
        public void run() {
            this.asyncTimeoutFuture = scheduler.schedule(() -> {
                clients.remove(LongPollingClient.this);
                List<ConfigGroupEnum> changedGroups = compareChangedGroup((HttpServletRequest) asyncContext.getRequest());
                sendResponse(changedGroups);
            }, timeoutTime, TimeUnit.MILLISECONDS);
            clients.add(this);
        }

        void sendResponse(final List<ConfigGroupEnum> changedGroups) {
            // cancel scheduler
            if (null != asyncTimeoutFuture) {
                asyncTimeoutFuture.cancel(false);
            }
            generateResponse((HttpServletResponse) asyncContext.getResponse(), changedGroups);
            asyncContext.complete();
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;通过的上面的分析，我们知道了一个数据变化监听的大致流程：

- 判断是否有数据变更：
  - 有变更：直接返回
  - 没有变更：启动任务等待

&ensp;&ensp;&ensp;&ensp;它有个asyncContext,如果超时了会返回空吗？

### 事件发布：Cache更新
&ensp;&ensp;&ensp;&ensp;在上面的探索中，没有和事件变化发布结合起来，下面我们来探索下从Controllers接口-->事件发布-->HTTP监听处理的流程

&ensp;&ensp;&ensp;&ensp;首先我们在上面的本地缓存更新函数中打上断点，查看其调用栈，但非常的可惜，根本没有触发......非常的奇怪

```java
public class ConfigDataCache {

    protected synchronized void update(final String md5, final long lastModifyTime) {
        this.md5 = md5;
        this.lastModifyTime = lastModifyTime;
    }
}
```

&ensp;&ensp;&ensp;&ensp;我们查看日志，找到我们刚才在后台页面中更新的事件日志

```text
o.d.s.a.l.AbstractDataChangedListener    : update config cache[PLUGIN], old: {group='PLUGIN', md5='5e40806d61fedeaf4f20be6a61c43991', lastModifyTime=1611713809914}, updated: {group='PLUGIN', md5='9166fd41ff37951bd871025178ebfac3', lastModifyTime=1611713823241}
```

&ensp;&ensp;&ensp;&ensp;根据日志，我们定位到相应的类和函数。需要注意的是，我们直接重启也能进入断点

&ensp;&ensp;&ensp;&ensp;从下面的代码中可以看出，已经收到了变化的数据，然后更新本地缓存

```java
public abstract class AbstractDataChangedListener implements DataChangedListener, InitializingBean {
    protected <T> void updateCache(final ConfigGroupEnum group, final List<T> data) {
        String json = GsonUtils.getInstance().toJson(data);
        ConfigDataCache newVal = new ConfigDataCache(group.name(), json, Md5Utils.md5(json), System.currentTimeMillis());
        // 更新本地缓存
        ConfigDataCache oldVal = CACHE.put(newVal.getGroup(), newVal);
        log.info("update config cache[{}], old: {}, updated: {}", group, oldVal, newVal);
    }
```

&ensp;&ensp;&ensp;&ensp;我们跟踪其调用栈，触发了下面代码中的Cache更新函数

&ensp;&ensp;&ensp;&ensp;然后来到一个奇怪的afterPropertiesSet

```java
public class HttpLongPollingDataChangedListener extends AbstractDataChangedListener {
    protected void updatePluginCache() {
        this.updateCache(ConfigGroupEnum.APP_AUTH, appAuthService.listAll());
    }

    // Admin重启后就触发，根据下面的内容，可以看出这就是一个初始化的函数，得到数据刷新本地缓存
    public final void afterPropertiesSet() {
        updateAppAuthCache();
        updatePluginCache();
        updateRuleCache();
        updateSelectorCache();
        updateMetaDataCache();
        afterInitialize();
    }

    // 这个线程启动了一个任务，定时周期执行指定的任务
    protected void afterInitialize() {
        long syncInterval = httpSyncProperties.getRefreshInterval().toMillis();
        // Periodically check the data for changes and update the cache
        scheduler.scheduleWithFixedDelay(() -> {
            log.info("http sync strategy refresh config start.");
            try {
                this.refreshLocalCache();
                log.info("http sync strategy refresh config success.");
            } catch (Exception e) {
                log.error("http sync strategy refresh config error!", e);
            }
        }, syncInterval, syncInterval, TimeUnit.MILLISECONDS);
        log.info("http sync strategy refresh interval: {}ms", syncInterval);
    }

    private void refreshLocalCache() {
        this.updateAppAuthCache();
        this.updatePluginCache();
        this.updateRuleCache();
        this.updateSelectorCache();
        this.updateMetaDataCache();
    }
}
```

&ensp;&ensp;&ensp;&ensp;在上面的函数中，我们发现它是初始化的操作，进行本地缓存的更新

&ensp;&ensp;&ensp;&ensp;在上面的探索中，还是没有找到事件发布之类的流程，我们直接在HttpLongPollingDataChangedListener的下面函数中打上断点，然后在后台改变插件状态

```java
public class HttpLongPollingDataChangedListener extends AbstractDataChangedListener {
    @Override
    protected void afterPluginChanged(final List<PluginData> changed, final DataEventTypeEnum eventType) {
        scheduler.execute(new DataChangeTask(ConfigGroupEnum.PLUGIN));
    }
}
```

&ensp;&ensp;&ensp;&ensp;execute后面再看，跟踪调用栈来到下面的代码

&ensp;&ensp;&ensp;&ensp;可以看到是先更新缓存，然后再执行后面afterPluginChanged的execute，调用栈后面就是熟悉的：DataChangedEventDispatcher：：onApplicationEvent、PluginServiceImpl：：createOrUpdate、PluginController：：updatePlugin，这里就不再赘述了

```java
public abstract class AbstractDataChangedListener implements DataChangedListener, InitializingBean {
        @Override
    public void onPluginChanged(final List<PluginData> changed, final DataEventTypeEnum eventType) {
        if (CollectionUtils.isEmpty(changed)) {
            return;
        }
        this.updatePluginCache();
        this.afterPluginChanged(changed, eventType);
    }

    protected void updatePluginCache() {
        this.updateCache(ConfigGroupEnum.PLUGIN, pluginService.listAll());
    }

    protected <T> void updateCache(final ConfigGroupEnum group, final List<T> data) {
        String json = GsonUtils.getInstance().toJson(data);
        ConfigDataCache newVal = new ConfigDataCache(group.name(), json, Md5Utils.md5(json), System.currentTimeMillis());
        ConfigDataCache oldVal = CACHE.put(newVal.getGroup(), newVal);
        log.info("update config cache[{}], old: {}, updated: {}", group, oldVal, newVal);
    }
}
```

&ensp;&ensp;&ensp;&ensp;我们来看一下execute函数的具体细节：

```java
public class HttpLongPollingDataChangedListener extends AbstractDataChangedListener {
    class DataChangeTask implements Runnable {

        @Override
        public void run() {
            // 遍历所有的client，然后发送变化的groupKey（数据类型）
            for (Iterator<LongPollingClient> iter = clients.iterator(); iter.hasNext();) {
                LongPollingClient client = iter.next();
                iter.remove();
                client.sendResponse(Collections.singletonList(groupKey));
                log.info("send response with the changed group,ip={}, group={}, changeTime={}", client.ip, groupKey, changeTime);
            }
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;在上面的线程中启动任务，循环遍历client列表，进行响应的发送，感觉和前面的定时监听任务有关，我们在回过头看看那段代码：

```java
public class HttpLongPollingDataChangedListener extends AbstractDataChangedListener {

    class LongPollingClient implements Runnable {
        @Override
        public void run() {
            this.asyncTimeoutFuture = scheduler.schedule(() -> {
                // 先将client从列表中移除
                clients.remove(LongPollingClient.this);
                List<ConfigGroupEnum> changedGroups = compareChangedGroup((HttpServletRequest) asyncContext.getRequest());
                sendResponse(changedGroups);
            }, timeoutTime, TimeUnit.MILLISECONDS);
            // 超时后将其加入列表中？
            clients.add(this);
        }

        void sendResponse(final List<ConfigGroupEnum> changedGroups) {
            // cancel scheduler
            if (null != asyncTimeoutFuture) {
                asyncTimeoutFuture.cancel(false);
            }
            generateResponse((HttpServletResponse) asyncContext.getResponse(), changedGroups);
            asyncContext.complete();
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;猜测流程是这样的：

- 1.如果客户端监听请求过来
  - 首先开一个超时任务，判断数据是否有变化，如果有变化，则进行响应的发送
  - 将client加入列表，用于后面的数据变化时，能进行推送（移除应该是为了避免重复）

- 2.事件进行发布，遍历列表中的client，返回变化响应给客户端

## 总结
&ensp;&ensp;&ensp;&ensp;比较其他三个同步方式，HTTP的是很复杂，用了一种自己以前没有见过的方式实现的，感觉又学到了新东西

&ensp;&ensp;&ensp;&ensp;Admin的HTTP长轮询的数据同步流程如下：

- 1.初始化
  - 进行本地缓存更新
  - 开启一个定时周期任务，定时刷新本地缓存
- 2.数据更新处理流程
  - controllers ：接口调用
  - Service ：服务调用
  - 事件发布
  - 数据更新：首先更新本地缓存；启动一个任务返回监听结果给等待中的客户端（通过监听端口得到变化，然后请求数据）

&ensp;&ensp;&ensp;&ensp;此外还有一个可能可以提PR的点，下面的函数似乎是没有用的(而且也没有搜索到那里使用，初始化和更新数据都没有能触发），在更新缓存的过程中基本都是new一个新的。也不排除其他地方有，但目前是没有找到，如果能确认的话，可以提一个PR进行删除

```java
public class ConfigDataCache {

    protected synchronized void update(final String md5, final long lastModifyTime) {
        this.md5 = md5;
        this.lastModifyTime = lastModifyTime;
    }
}
```

## Soul网关源码解析文章列表
### 掘金
- [Soul网关源码解析（一） 概览](https://juejin.cn/post/6917864624423436296)
- [Soul网关源码解析（二）代码初步运行](https://juejin.cn/post/6917865804121767944)
- [Soul网关源码解析（三）请求处理概览](https://juejin.cn/post/6917866538712334343)
- [Soul网关源码解析（四）Dubbo请求概览](https://juejin.cn/post/6917867369909977102)
- [Soul网关源码解析（五）请求类型探索](https://juejin.cn/post/6918575905962983438)
- [Soul网关源码解析（六）Sofa请求处理概览](https://juejin.cn/post/6918736260467015693)
- [Soul网关源码解析（七）限流插件初探](https://juejin.cn/post/6919348164944232455/)
- [Soul网关源码解析（八）路由匹配初探](https://juejin.cn/post/6919774553241550855/)
- [Soul网关源码解析（九）插件配置加载初探](https://juejin.cn/post/6920074307590684685/)
- [Soul网关源码解析（十）自定义简单插件编写](https://juejin.cn/post/6920142348617777166)
- [Soul网关源码解析（十一）请求处理小结](https://juejin.cn/post/6920596034171174925)
- [Soul网关源码解析（十二）数据同步初探](https://juejin.cn/post/6920596173925384206)
- [Soul网关源码解析（十三）Websocket同步数据-Bootstrap端](https://juejin.cn/post/6920596028505178125)
- [Soul网关源码解析（十四）HTTP数据同步-Bootstrap端](https://juejin.cn/post/6920597298674302983)
- [Soul网关源码解析（十五）Zookeeper数据同步-Bootstrap端](https://juejin.cn/post/6920764643967238151)
- [Soul网关源码解析（十六）Nacos数据同步示例运行](https://juejin.cn/post/6921170233868845064)
- [Soul网关源码解析（十七）Nacos数据同步解析-Bootstrap端](https://juejin.cn/post/6921325882753695757/)
- [Soul网关源码解析（十八）Zookeeper数据同步初探-Admin端](https://juejin.cn/post/6921495273122463751/)
- [Soul网关源码解析（十九）Nacos数据同步初始化修复-Admin端](https://juejin.cn/post/6921621915995996168/)
- [Soul网关源码解析（二十）Websocket数据同步-Admin端](https://juejin.cn/post/6921988280187617287/)

- [Soul网关源码阅读番外篇（一） HTTP参数请求错误](https://juejin.cn/post/6918947689564471309)