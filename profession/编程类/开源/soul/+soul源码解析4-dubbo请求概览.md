# Soul 网关源码解析（四）Dubbo请求概览
***
## 简介
&ensp;&ensp;&ensp;&ensp;本次启动一个dubbo服务示例，初步探索Soul网关源码的Dubbo请求处理流程

## 概览
&ensp;&ensp;&ensp;&ensp;接着上篇：[Soul网关源码解析（三）请求处理概览](https://juejin.cn/post/6917866538712334343)

&ensp;&ensp;&ensp;&ensp;我们来看一下Dubbo请求的代理转发和HTTP的有什么不一样的地方；经历了那些类；进行了那些操作

## 示例运行
### 环境配置
&ensp;&ensp;&ensp;&ensp;在Soul源码clone下来以后，有一个 soul-example 目录，这个就是示例工程，里面有很多的示例可以运行

&ensp;&ensp;&ensp;&ensp;可能初始文件夹不被IDEA识别为Java工程，我们需要点击改工程目录下的 pom.xml 文件，然后在右键，在菜单中选择：add as maven project

&ensp;&ensp;&ensp;&ensp;我们选择运行：soul-examples --> soul-examples-apache-dubbo-service

&ensp;&ensp;&ensp;&ensp;此次示例需要mysql和zookeeper，我们使用docker启动一下,记得修改 soul-admin 的数据库配置，其他的都不需要动，猜测zk有默认配置

```shell script
docker run -dit --name zk -p 2181:2181 zookeepe
docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:latest
```

### 测试运行
&ensp;&ensp;&ensp;&ensp;然后运行 Soul-admin、Soul-bootstrap,可以参考：[Soul网关源码解析（二）代码初步运行](https://juejin.cn/post/6917865804121767944)

&ensp;&ensp;&ensp;&ensp;运行soul-examples-apache-dubbo-service

&ensp;&ensp;&ensp;&ensp;登录 soul-admin 管理界面：http://localhost:9095/ ，账号和密码：admin 123456

&ensp;&ensp;&ensp;&ensp;进入界面：系统管理 --> 插件管理

&ensp;&ensp;&ensp;&ensp;在插件：dubbo，点击编辑，将状态修改为开启（是个开关图标）

&ensp;&ensp;&ensp;&ensp;开启成功后，可以在Soul-Bootstrap中看到相关的日志，大致如下：

```java
o.d.s.p.a.d.c.ApplicationConfigCache     : init aliaba dubbo reference success there meteData is :MetaData
o.d.s.p.a.d.c.ApplicationConfigCache     : init aliaba dubbo reference success there meteData is :MetaData
o.d.s.p.a.d.c.ApplicationConfigCache     : init aliaba dubbo reference success there meteData is :MetaData
```

&ensp;&ensp;&ensp;&ensp;进入Soul-admin管理界面：插件管理 --> dubbo ，我们可以看到很多的配置，这些都是soul-examples-apache-dubbo-service的配置


&ensp;&ensp;&ensp;&ensp;随便找个查询的接口：http://localhost:9195/dubbo/findAll

&ensp;&ensp;&ensp;&ensp;使用浏览器访问得到的结果如下：

```json
{
    "code": 200,
    "message": "Access to success!",
    "data": {
        "name": "hello world Soul Apache, findAll",
        "id": "-1206394682"
    }
}
```

&ensp;&ensp;&ensp;&ensp;很好，已经成功跑通示例，接下来，进入源码进行debug

## 源码分析
&ensp;&ensp;&ensp;&ensp;基于上篇，我们已经知道了一个HTTP请求的初步处理流程：[soul源码阅读3-请求处理概览.md](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB3-%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%A6%82%E8%A7%88.md)

&ensp;&ensp;&ensp;&ensp;基于上篇的熟悉度，我们就开始看看Dubbo的处理流程和Http的有什么区别，顺带看一看各个Plugin都干了啥

&ensp;&ensp;&ensp;&ensp;Plugins的链式处理核心类是：**SoulWebHandler**，我们就从这里开始打上断点，逐步进入每个Plugin进行查看

```java
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
```

### GlobalPlugin
&ensp;&ensp;&ensp;&ensp;代码看的不是很懂，猜测有下面的功能：

- 获取请求头的 upgrade，此时为null，进入逻辑
- 另外的分支的MultiValueMap有点像文件上传之类需要的，后面有时间验证一下
- 功能是修改了SoulContext

&ensp;&ensp;&ensp;&ensp;这个不是此次分析的目标，到这就跳过


### SignPlugin
&ensp;&ensp;&ensp;&ensp;这个应该是一个认证相关的插件，但在获取插件信息，没有开启，不进入逻辑，直接跳过，没有进入具体执行

&ensp;&ensp;&ensp;&ensp;这里发现有的能需要进入函数：execute，有的直接进入：doExecute。稍微有点好奇，就看了下原因，大致是直接继承SoulPlugin就直接进入doExecute,而 AbstractSoulPlugin需要进入execute，这里不是重点，就不继续研究这个了，有个大概了解即可

### WafPlugin、RateLimiterPlugin、HystrixPlugin、Resilience4JPlugin
&ensp;&ensp;&ensp;&ensp;上面几个都是获取插件信息，没有开启，不进入逻辑，直接跳过

### DividePlugin
&ensp;&ensp;&ensp;&ensp;这里是true，有点和自己想的不一样，查看管理界面看到divide是开启的，稍微挖一下这个

&ensp;&ensp;&ensp;&ensp;定位到关键函数：Boolean skip = plugin.skip(exchange)，判断是否可以跳过，我们看下具体的代码：

```java
    public Boolean skip(final ServerWebExchange exchange) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        return !Objects.equals(Objects.requireNonNull(soulContext).getRpcType(), RpcTypeEnum.HTTP.getName());
    }
```

&ensp;&ensp;&ensp;&ensp;好像就是判断类型来返回是否跳过，其他的plugins的判断逻辑，至于这些都是都是怎么来的，今天就不分析了，保留体力，留待下次进行分析

### WebClientPulugin、WebsocketPlugin
&ensp;&ensp;&ensp;&ensp;skip 为 true，直接跳过,跳过的判断的和DividePlugin基本相同，通过类型判断

### BodyParamPlugin
&ensp;&ensp;&ensp;&ensp;对soulContext进行了操作，和GlobalPlugin有一些联动，还看到了MediaType等关键字，感觉很像文件上传之类的，但细节不太清楚，这个也不是本次分析的目的，不要陷入细节，这次放过它，下次有时间再仔细研究

### AlibabaDubblePlugin
&ensp;&ensp;&ensp;&ensp;这个插件是本次的核心，大致干了下面三个事情

- 进入了并且匹配上了规则，发现这个plugin是dubbo总插件，同样能处理Apache dubbo，也就是他们是相同的或者复用的
- 进入到doExecute函数：获取body，soulContext，查看soulContext发现已经有方法、路径等信息（猜测是前面加的），获得关键的metaData
- 获取请求的结果，并且放到exchange中：Object result = alibabaDubboProxyService.genericInvoker(body, metaData)

&ensp;&ensp;&ensp;&ensp;代码大致如下：

```java
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        String body = exchange.getAttribute(Constants.DUBBO_PARAMS);
        SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        // dubbo 请求的相关数据
        MetaData metaData = exchange.getAttribute(Constants.META_DATA);
        if (!checkMetaData(metaData)) {
            assert metaData != null;
            log.error(" path is :{}, meta data have error.... {}", soulContext.getPath(), metaData.toString());
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            Object error = SoulResultWrap.error(SoulResultEnum.META_DATA_ERROR.getCode(), SoulResultEnum.META_DATA_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        if (StringUtils.isNoneBlank(metaData.getParameterTypes()) && StringUtils.isBlank(body)) {
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            Object error = SoulResultWrap.error(SoulResultEnum.DUBBO_HAVE_BODY_PARAM.getCode(), SoulResultEnum.DUBBO_HAVE_BODY_PARAM.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        // 获取请求结果
        Object result = alibabaDubboProxyService.genericInvoker(body, metaData);
        if (Objects.nonNull(result)) {
            // 将结果放到exchange中
            exchange.getAttributes().put(Constants.DUBBO_RPC_RESULT, result);
        } else {
            exchange.getAttributes().put(Constants.DUBBO_RPC_RESULT, Constants.DUBBO_RPC_RESULT_EMPTY);
        }
        exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
        return chain.execute(exchange);
    }
```

&ensp;&ensp;&ensp;&ensp;继续跟踪查看获取result的那个函数，通过下面的代码可以明显看出确实是rpc调用，并得到结果

```java
    public Object genericInvoker(final String body, final MetaData metaData) throws SoulException {
        // 通过rpc和dubbo的相关知识，这里的reference相当于consumer，metaData.getPath()==/dubbo/findAll
        // ApplicationConfigCache.getInstance().get 的逻辑感觉比较复杂，先不看了，大体感觉是初始化的时候使用 path 作为了 key
        ReferenceConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
        if (Objects.isNull(reference) || StringUtils.isEmpty(reference.getInterface())) {
            ApplicationConfigCache.getInstance().invalidate(metaData.getPath());
            reference = ApplicationConfigCache.getInstance().initRef(metaData);
        }
        // 这段很像RPC Demo中的获取字节码生成的对象
        GenericService genericService = reference.get();
        try {
            Pair<String[], Object[]> pair;
            if (ParamCheckUtils.dubboBodyIsEmpty(body)) {
                pair = new ImmutablePair<>(new String[]{}, new Object[]{});
            } else {
                pair = dubboParamResolveService.buildParameter(body, metaData.getParameterTypes());
            }
            // 传入方法名、参数名、参数？Dubbo还没用的熟练，暂时这样猜一猜，不影响此次分析的大局
            return genericService.$invoke(metaData.getMethodName(), pair.getLeft(), pair.getRight());
        } catch (GenericException e) {
            log.error("dubbo invoker have exception", e);
            throw new SoulException(e.getExceptionMessage());
        }
    }
```

&ensp;&ensp;&ensp;&ensp;对reference.get()，有点好奇，稍微看下 ReferenceConfig

```java
    public synchronized T get() {
        // 判断是否能用
        if (this.destroyed) {
            throw new IllegalStateException("Already destroyed!");
        } else {
            // 有延迟加载的作用
            if (this.ref == null) {
                this.init();
            }

            return this.ref;
        }
    }
```

&ensp;&ensp;&ensp;&ensp;看到它这用法还是挺巧妙的，有一个延迟加载的效果，好像可以按照这个思路去改一改自己的RPC Demo的客户端代理生成，哈哈

### MonitorPlugin
&ensp;&ensp;&ensp;&ensp;获取插件信息，没有开启，不进入逻辑，直接跳过

### WebClientResponsePlugin
&ensp;&ensp;&ensp;&ensp;skip = true，直接跳过

### DubboResponsePlugin
&ensp;&ensp;&ensp;&ensp;看名字就知道是个核心类，我们在then后执行的代码段打个断点（这种需要单独打个端口，后面才能跳进去）

```java
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        return chain.execute(exchange).then(Mono.defer(() -> {
            // 从exchange中拿到结果
            final Object result = exchange.getAttribute(Constants.DUBBO_RPC_RESULT);
            try {
                if (Objects.isNull(result)) {
                    Object error = SoulResultWrap.error(SoulResultEnum.SERVICE_RESULT_ERROR.getCode(), SoulResultEnum.SERVICE_RESULT_ERROR.getMsg(), null);
                    return WebFluxResultUtils.result(exchange, error);
                }
                Object success = SoulResultWrap.success(SoulResultEnum.SUCCESS.getCode(), SoulResultEnum.SUCCESS.getMsg(), JsonUtils.removeClass(result));
                // 进入后使用之前熟悉的：exchange.getResponse().writeWith，返回响应给客户端
                return WebFluxResultUtils.result(exchange, success);
            } catch (SoulException e) {
                return Mono.empty();
            }
        }));
    }
```

&ensp;&ensp;&ensp;&ensp;执行完plugins链后进入到这个断点：大致是获取结果result，加工得到success

&ensp;&ensp;&ensp;&ensp;然后WebFluxResultUtils.result(exchange, success)，进入后是：exchange.getResponse().writeWith，返回响应给客户端

&ensp;&ensp;&ensp;&ensp;到这里，一个Dubbo请求的处理流程大致展示在我们面前了，有了前面几篇的分析基础，还是挺流畅的

## 总结
&ensp;&ensp;&ensp;&ensp;此次分析验证了在上篇中的一些猜想，也得到了一些新的东西，请求处理的大致流程图如下：

![](./picture/Soulprocessfirst.png)

- HttpServerOperations : 明显的netty的请求接收的地方，请求入口
- ReactorHttpHandlerAdapter ：生成response和request
- HttpWebHandlerAdapter ：exchange 的生成
- FilteringWebHandler : filter 操作
- SoulWebHandler ：plugins调用

&ensp;&ensp;&ensp;&ensp;可以参考下上篇分析： [Soul 源码阅读（三）HTTP请求处理概览](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB3-%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%A6%82%E8%A7%88.md)

&ensp;&ensp;&ensp;&ensp;这篇中我们详细分析各个Dubbo 经过的 Plugin 的相关处理，有些复杂的就没有看了，但也得到我们此次分析想要的结果，脑海中对一个RPC请求如果进行处理，设计那些组件有了一定的了解，基于这些了解也能进行更近一步的分析

&ensp;&ensp;&ensp;&ensp;我们收获和HTTP请求调用、RPC请求调用、Websocket请求调用是互斥的，通过判断请求的类型来进行选择其中一个，而HTTP相应、RPC响应都会转为HTTP响应，然后返回给客户端，exchange里面估计是绑定了一个socket，然后直接调用即可（Netty网关Demo又可以进行借鉴了）

&ensp;&ensp;&ensp;&ensp;此次分析有了下面新的疑问：

- Websocket的响应的返回时怎样的？是长连接吗？因为没有看到Websocket响应相关的处理类
- exchange中的请求类型是怎么绑定，如何生成的？
- 限流等插件也是需要进行匹配吗？具体执行逻辑是怎样的？

&ensp;&ensp;&ensp;&ensp;上面这些问题就留待以后分析了，慢慢来才能可持续发展

## Soul网关源码解析文章列表
### 掘金
#### 了解与初步运行
- [Soul网关源码解析（一） 概览](https://juejin.cn/post/6917864624423436296)
- [Soul网关源码解析（二）代码初步运行](https://juejin.cn/post/6917865804121767944)

#### 请求处理流程解析
- [Soul网关源码解析（三）请求处理概览](https://juejin.cn/post/6917866538712334343)
- [Soul网关源码解析（四）Dubbo请求概览](https://juejin.cn/post/6917867369909977102)
- [Soul网关源码解析（五）请求类型探索](https://juejin.cn/post/6918575905962983438)
- [Soul网关源码解析（六）Sofa请求处理概览](https://juejin.cn/post/6918736260467015693)
- [Soul网关源码解析（七）限流插件初探](https://juejin.cn/post/6919348164944232455/)
- [Soul网关源码解析（八）路由匹配初探](https://juejin.cn/post/6919774553241550855/)
- [Soul网关源码解析（九）插件配置加载初探](https://juejin.cn/post/6920074307590684685/)
- [Soul网关源码解析（十）自定义简单插件编写](https://juejin.cn/post/6920142348617777166)
- [Soul网关源码解析（十一）请求处理小结](https://juejin.cn/post/6920596034171174925)

#### 数据同步解析
- [Soul网关源码解析（十二）数据同步初探](https://juejin.cn/post/6920596173925384206)
- [Soul网关源码解析（十三）Websocket同步数据-Bootstrap端](https://juejin.cn/post/6920596028505178125)
- [Soul网关源码解析（十四）HTTP数据同步-Bootstrap端](https://juejin.cn/post/6920597298674302983)
- [Soul网关源码解析（十五）Zookeeper数据同步-Bootstrap端](https://juejin.cn/post/6920764643967238151)
- [Soul网关源码解析（十六）Nacos数据同步示例运行](https://juejin.cn/post/6921170233868845064)
- [Soul网关源码解析（十七）Nacos数据同步解析-Bootstrap端](https://juejin.cn/post/6921325882753695757/)
- [Soul网关源码解析（十八）Zookeeper数据同步初探-Admin端](https://juejin.cn/post/6921495273122463751/)
- [Soul网关源码解析（十九）Nacos数据同步初始化修复-Admin端](https://juejin.cn/post/6921621915995996168/)
- [Soul网关源码解析（二十）Websocket数据同步-Admin端](https://juejin.cn/post/6921988280187617287/)
- [Soul网关源码解析（二十一）HTTP长轮询数据同步-Admin端](https://juejin.cn/post/6922301585288593416/)
- [Soul网关源码解析（二十二）数据同步小结](https://juejin.cn/post/6922584596810825735/)

#### Soul-Client模块
- [Soul网关源码解析（二十三）SoulSpringMvcClient注解](https://juejin.cn/post/6922643958455599111)
- [Soul网关源码解析（二十四）SoulDubboClient注解](https://juejin.cn/post/6922722161702469640/)
- [Soul网关源码解析（二十五）Soul-Client模块小结](https://juejin.cn/post/6922745260435046408/)

#### 总览
- [Soul网关源码解析（二十六）初步总览](https://juejin.cn/post/6922960265663217678/)

#### 番外
- [Soul网关源码阅读番外篇（一） HTTP参数请求错误](https://juejin.cn/post/6918947689564471309)