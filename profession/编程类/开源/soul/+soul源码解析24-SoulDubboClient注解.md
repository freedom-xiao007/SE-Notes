# Soul网关源码解析（二十四）SoulDubboClient注解
***
## 简介
&ensp;&ensp;&ensp;&ensp;本篇文章来探索下Soul网关Apache Dubbo客户端的注册处理流程

## 概览
&ensp;&ensp;&ensp;&ensp;有了上一篇的基础：[Soul网关源码解析（二十三）SoulSpringMvcClient注解](https://juejin.cn/post/6922643958455599111)

&ensp;&ensp;&ensp;&ensp;我们直接在Soul-Client模块下定位到：soul-client-apache-dubbo，找到进行具体处理的类：ApacheDubboServiceBeanPostProcessor

&ensp;&ensp;&ensp;&ensp;我们看下是如何获取到Bean的？如果判断是Soul-Client-Apache-Dubbo的？注册了那些信息？

&ensp;&ensp;&ensp;&ensp;下面开始分析：

## 源码Debug
### 如何获取到Bean
&ensp;&ensp;&ensp;&ensp;直接定位到处理类如下，onApplicationEvent方法是入口，那应该是从这里开始像HTTP那样获取Bean的

&ensp;&ensp;&ensp;&ensp;首先看下继承的类，可以看到和HTTP不同，这里是：ApplicationListener<ContextRefreshedEvent>

&ensp;&ensp;&ensp;&ensp;由于不熟悉，我们搜索下它是干嘛的，参考链接是：[ApplicationListener和ContextRefreshedEvent](https://www.jianshu.com/p/4bd3f68cb179)

```
在IOC的容器的启动过程，当所有的bean都已经处理完成之后，spring ioc容器会有一个发布事件的动作。从 AbstractApplicationContext 的源码中就可以看出
当ioc容器加载处理完相应的bean之后，也给我们提供了一个机会（先有InitializingBean，后有ApplicationListener<ContextRefreshedEvent>），可以去做一些自己想做的事。其实这也就是spring ioc容器给提供的一个扩展的地方。我们可以这样使用这个扩展机制。

作者：AmeeLove
链接：https://www.jianshu.com/p/4bd3f68cb179
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

&ensp;&ensp;&ensp;&ensp;大致就是在所有容器都初始化完成以后，会触发执行

&ensp;&ensp;&ensp;&ensp;我们在下面的代码中打断点进行调试，发现:contextRefreshedEvent.getApplicationContext().getBeansOfType(ServiceBean.class),执行完以后就可以得到我们示例中的两个Impl服务，感觉挺便利的。这个感觉需要对Dubbo有一些熟悉，现在又学到了一手

&ensp;&ensp;&ensp;&ensp;获取以后遍历开启线程去进行注册，获取Bean的过程大致就是这样了

```java
public class ApacheDubboServiceBeanPostProcessor implements ApplicationListener<ContextRefreshedEvent> {

	// DubboConfig(adminUrl=http://localhost:9095, contextPath=/dubbo, appName=dubbo)
    private DubboConfig dubboConfig;

    private ExecutorService executorService;

	// http://localhost:9095/soul-client/dubbo-register
    private final String url;

    public ApacheDubboServiceBeanPostProcessor(final DubboConfig dubboConfig) {
        String contextPath = dubboConfig.getContextPath();
        String adminUrl = dubboConfig.getAdminUrl();
        if (StringUtils.isEmpty(contextPath)
                || StringUtils.isEmpty(adminUrl)) {
            throw new RuntimeException("apache dubbo client must config the contextPath, adminUrl");
        }
        this.dubboConfig = dubboConfig;
        url = dubboConfig.getAdminUrl() + "/soul-client/dubbo-register";
        executorService = new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>());
    }

    @Override
    public void onApplicationEvent(final ContextRefreshedEvent contextRefreshedEvent) {
        if (Objects.nonNull(contextRefreshedEvent.getApplicationContext().getParent())) {
            return;
        }
        // Fix bug(https://github.com/dromara/soul/issues/415), upload dubbo metadata on ContextRefreshedEvent
        Map<String, ServiceBean> serviceBean = contextRefreshedEvent.getApplicationContext().getBeansOfType(ServiceBean.class);
        for (Map.Entry<String, ServiceBean> entry : serviceBean.entrySet()) {
            executorService.execute(() -> handler(entry.getValue()));
        }
    }
}
```

### 判断是否需要注册
&ensp;&ensp;&ensp;&ensp;我们直接来到具体处理的handler函数

&ensp;&ensp;&ensp;&ensp;可以看到其直接从bean获取到类信息，而且如果是发射代理类的话，要获取到原来被代理的类（这里有些奇怪，为啥不能直接使用代理类，是方法生成后有什么不同吗？后面去探索一下）

&ensp;&ensp;&ensp;&ensp;这里没有看到像HTTP那样的通配符了，Dubbo好像不能这样，这个感觉是个值得注意的点，感觉通配的话在从URL转Dubbo的路径的时候不好转，所有没有提供这方面的机制

&ensp;&ensp;&ensp;&ensp;最后遍历类中的所有方法，如果有SoulDubboClient注解，将其接口注入到Admin中

```java
public class ApacheDubboServiceBeanPostProcessor implements ApplicationListener<ContextRefreshedEvent> {

    private void handler(final ServiceBean serviceBean) {
		// clazz == org.dromara.soul.examples.apache.dubbo.service.impl.DubboTestServiceImpl
        Class<?> clazz = serviceBean.getRef().getClass();
		// 这应该是得到反射代理生成的原本的类吧（为啥不能直接使用代理类呢？）
        if (ClassUtils.isCglibProxyClass(clazz)) {
            String superClassName = clazz.getGenericSuperclass().getTypeName();
            try {
                clazz = Class.forName(superClassName);
            } catch (ClassNotFoundException e) {
                log.error(String.format("class not found: %s", superClassName));
                return;
            }
        }
		// 遍历所有的方法，如果有相关的注解，则注册到Admin中
        final Method[] methods = ReflectionUtils.getUniqueDeclaredMethods(clazz);
        for (Method method : methods) {
			// public org.dromara.soul.examples.dubbo.api.entity.DubboTest org.dromara.soul.examples.apache.dubbo.service.impl.DubboTestServiceImpl.insert(org.dromara.soul.examples.dubbo.api.entity.DubboTest)
            SoulDubboClient soulDubboClient = method.getAnnotation(SoulDubboClient.class);
            if (Objects.nonNull(soulDubboClient)) {
                RegisterUtils.doRegister(buildJsonParams(serviceBean, soulDubboClient, method), url, RpcTypeEnum.DUBBO);
            }
        }
    }
}
```

### 注册信息
&ensp;&ensp;&ensp;&ensp;下面是构造请求相关的函数，从其中可以看出，注册了选择器和规则相关的信息。在其中的Ext中还有分组、版本、负载均衡等信息

&ensp;&ensp;&ensp;&ensp;我们没有看到像HTTP的那些IP和端口之类的信息，但联想到RPC的相关知识，大致知道Dubbo已经将其注册到Zookeeper中了

```java
public class ApacheDubboServiceBeanPostProcessor implements ApplicationListener<ContextRefreshedEvent> {

	// DubboConfig(adminUrl=http://localhost:9095, contextPath=/dubbo, appName=dubbo)
    private DubboConfig dubboConfig;

    private ExecutorService executorService;

	// http://localhost:9095/soul-client/dubbo-register
    private final String url;

    private String buildJsonParams(final ServiceBean serviceBean, final SoulDubboClient soulDubboClient, final Method method) {
        String appName = dubboConfig.getAppName();
        if (StringUtils.isEmpty(appName)) {
            appName = serviceBean.getApplication().getName();
        }
        String path = dubboConfig.getContextPath() + soulDubboClient.path();
        String desc = soulDubboClient.desc();
		// org.dromara.soul.examples.dubbo.api.service.DubboTestService
        String serviceName = serviceBean.getInterface();
        String configRuleName = soulDubboClient.ruleName();
		// /dubbo/findAll
        String ruleName = ("".equals(configRuleName)) ? path : configRuleName;
        String methodName = method.getName();
        Class<?>[] parameterTypesClazz = method.getParameterTypes();
        String parameterTypes = Arrays.stream(parameterTypesClazz).map(Class::getName)
                .collect(Collectors.joining(","));
        MetaDataDTO metaDataDTO = MetaDataDTO.builder()
                .appName(appName)
                .serviceName(serviceName)
                .methodName(methodName)
                .contextPath(dubboConfig.getContextPath())
                .path(path)
                .ruleName(ruleName)
                .pathDesc(desc)
                .parameterTypes(parameterTypes)
                .rpcExt(buildRpcExt(serviceBean))
                .rpcType("dubbo")
                .enabled(soulDubboClient.enabled())
                .build();
        return OkHttpTools.getInstance().getGson().toJson(metaDataDTO);

    }

    private String buildRpcExt(final ServiceBean serviceBean) {
        MetaDataDTO.RpcExt build = MetaDataDTO.RpcExt.builder()
                .group(StringUtils.isNotEmpty(serviceBean.getGroup()) ? serviceBean.getGroup() : "")
                .version(StringUtils.isNotEmpty(serviceBean.getVersion()) ? serviceBean.getVersion() : "")
                .loadbalance(StringUtils.isNotEmpty(serviceBean.getLoadbalance()) ? serviceBean.getLoadbalance() : Constants.DEFAULT_LOADBALANCE)
                .retries(Objects.isNull(serviceBean.getRetries()) ? Constants.DEFAULT_RETRIES : serviceBean.getRetries())
                .timeout(Objects.isNull(serviceBean.getTimeout()) ? Constants.DEFAULT_CONNECT_TIMEOUT : serviceBean.getTimeout())
                .url("")
                .build();
        return OkHttpTools.getInstance().getGson().toJson(build);
    }
}
```

### Apache Dubbo 注册调用梳理
&ensp;&ensp;&ensp;&ensp;下面是最终发送的请求信息，不像HTTP那样直观，如果没有接触过RPC，可以感觉有点懵，这里梳理一下RPC相关的注册调用流程

```json
{
	"appName": "dubbo",
	"contextPath": "/dubbo",
	"path": "/dubbo/insert",
	"pathDesc": "Insert a row of data",
	"rpcType": "dubbo",
	"serviceName": "org.dromara.soul.examples.dubbo.api.service.DubboTestService",
	"methodName": "insert",
	"ruleName": "/dubbo/insert",
	"parameterTypes": "org.dromara.soul.examples.dubbo.api.entity.DubboTest",
	"rpcExt": "{\"group\":\"\",\"version\":\"\",\"loadbalance\":\"random\",\"retries\":2,\"timeout\":10000,\"url\":\"\"}",
	"enabled": true
}
```

&ensp;&ensp;&ensp;&ensp;一个Dubbo的注册和调用流程图如下：

![](./picture/soul-dubbo.png)

&ensp;&ensp;&ensp;&ensp;注册和调用都有可能同时发生，也就是路由Soul网关的路由时可以动态变化的

&ensp;&ensp;&ensp;&ensp;当有新的服务进行注册或服务更新时，就会将数据发给Zookeeper

&ensp;&ensp;&ensp;&ensp;Boostrap那边有一个Dubbo的客户端之类的东西，会受到Zookeeper变化的服务信息，实时进行更新

&ensp;&ensp;&ensp;&ensp;当请求进行访问时，Soul网关直接调用Dubbo的客户端，让它去访问后台的Dubbo服务，拿到结果后返回即可

## 总结
&ensp;&ensp;&ensp;&ensp;本篇对Soul-Client的Apache Dubbo模块进行探索，发现和HTTP的注册流程大致相同：

- 1.获取对应的Bean（类）：
  - HTTP是通过在每个Bean构造完成后进行操作，这样比Dubbo可能要费时
  - Dubbo在所有Bean构造完成后进行操作，直接能获取Dubbo相关的Bean

- 2.注册判断：两者都是通过判断是否满足相应的注解

- 3.注册信息：
  - HTTP没有中间层，所有必须要有额外的IP和端口信息
  - Dubbo有Zookeeper中间层，Boostrap有个Dubbo客户端，通过Zookeeper能得到后台信息，就没有传入额外的IP和端口信息

&ensp;&ensp;&ensp;&ensp;最后留一个疑问：为啥不能使用发射代理进行注册？

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

#### 番外
- [Soul网关源码阅读番外篇（一） HTTP参数请求错误](https://juejin.cn/post/6918947689564471309)
