# Soul网关源码解析（二十三）SoulSpringMvcClient注解
***
## 简介
&ensp;&ensp;&ensp;&ensp;本篇开始进入第三个模块：Soul-Client，这个模块负责自动注册路由相关的数据，本篇探索下HTTP服务注册的SoulSpringMvcClient注解

## 概览
&ensp;&ensp;&ensp;&ensp;大家如果使用过NGINX或者运行或soul，大致应该能了解路由转发规则都是需要配置的，在Soul网关中，有两种配置方式：一是手动在Soul-Admin后台管理界面进行配置；二是在后台服务中加上注解，让其自动注册配置

&ensp;&ensp;&ensp;&ensp;在前面文章中，我们也收到配置了一个divide插件的，需要配置选择器和规则，那在自动注册配置的注解中，是如果进行配置的？选择器、规则分别是如何获取和对应的，我们下面开始初步探索下

## 源码Debug
### 寻找切入点
&ensp;&ensp;&ensp;&ensp;这是第一次分析，还没有概念，我们先从运行日志查看下有没有蛛丝马迹。运行soul-examples-http模块，我们发现打印了下面的日志

```text
o.d.s.client.common.utils.RegisterUtils  : http client register success: {"appName":"http","context":"/http","path":"/http/order/findById","pathDesc":"Find by id","rpcType":"http","host":"172.21.160.1","port":8188,"ruleName":"/http/order/findById","enabled":true,"registerMetaData":false} 
o.d.s.client.common.utils.RegisterUtils  : http client register success: {"appName":"http","context":"/http","path":"/http/order/save","pathDesc":"Save order","rpcType":"http","host":"172.21.160.1","port":8188,"ruleName":"/http/order/save","enabled":true,"registerMetaData":false} 
```

&ensp;&ensp;&ensp;&ensp;通过上面这个日志可以看出，这是注册成功的信息，是在RegisterUtils中进行的，我们搜索这个类，查看下：

```java
public final class RegisterUtils {

    /**
     * call register api.
     *
     * @param json        request body
     * @param url         url
     * @param rpcTypeEnum rcp type
     */
    public static void doRegister(final String json, final String url, final RpcTypeEnum rpcTypeEnum) {
        try {
            // 调用第三方的HTTP客户端发送post请求，url是Soul-Admin的地址加接口
            String result = OkHttpTools.getInstance().post(url, json);
            if (AdminConstants.SUCCESS.equals(result)) {
                log.info("{} client register success: {} ", rpcTypeEnum.getName(), json);
            } else {
                log.error("{} client register error: {} ", rpcTypeEnum.getName(), json);
            }
        } catch (IOException e) {
            log.error("cannot register soul admin param, url: {}, request body: {}", url, json, e);
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;通过上面的函数，可以看到这个就是注册配置的最后一站了，我们在上面的函数打上断点，跟踪调用栈

&ensp;&ensp;&ensp;&ensp;重启后进入断点，来到下面的类，被下面的函数进行调用

```java
public class SpringMvcClientBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(@NonNull final Object bean, @NonNull final String beanName) throws BeansException {
        if (controller != null || restController != null || requestMapping != null) {
            SoulSpringMvcClient clazzAnnotation = AnnotationUtils.findAnnotation(bean.getClass(), SoulSpringMvcClient.class);
            String prePath = "";
            if (Objects.nonNull(clazzAnnotation)) {
                if (clazzAnnotation.path().indexOf("*") > 1) {
                    String finalPrePath = prePath;
                    executorService.execute(() -> RegisterUtils.doRegister(buildJsonParams(clazzAnnotation, finalPrePath), url,
                            RpcTypeEnum.HTTP));
                    return bean;
                }
                prePath = clazzAnnotation.path();
            }
        }
        return bean;
    }
}
```

&ensp;&ensp;&ensp;&ensp;继续往下发现调用栈没了，我们在上面的函数中打上断点，重启查看调用栈，发现是Spring相关的，BeanPostProcessor是关键，大致是会在Bean初始化后会触发运行

&ensp;&ensp;&ensp;&ensp;查阅参考的链接是：[Spring BeanPostProcessor接口使用](https://www.jianshu.com/p/e1c3c6e90e8a)

### 注册配置大致逻辑
&ensp;&ensp;&ensp;&ensp;在SpringMvcClientBeanPostProcessor的postProcessAfterInitialization函数上打上断点，会发现每个bean都会触发执行

&ensp;&ensp;&ensp;&ensp;之前还猜测注册配置信息是使用AOP方式执行的，但只才对了一般，应该是使用AOP并且是在初始化的时候进行注册配置的，感觉自己对Spring还有好多不了解，后面得搞一下了

&ensp;&ensp;&ensp;&ensp;在下面这个类中，初始化就有了Soul-Admin后台的url接口地址

&ensp;&ensp;&ensp;&ensp;我们看它的注册逻辑是：

- 1.如果类上有Controller、RestController、RequestMapping这三个注解之一，那将进行下一个Soul注解的判断注册
- 2.如果是在类上有Soul的相关注解
  - 如果注解是通配符，将所有接口都进行注册
  - 如果不是，先得到接口前缀，获取所有方法，将方法上有Soul相关注解的都给注册上去

```java
public class SpringMvcClientBeanPostProcessor implements BeanPostProcessor {

    private final ThreadPoolExecutor executorService;

    // http://localhost:9095/soul-client/springmvc-register
    private final String url;

    private final SoulSpringMvcConfig soulSpringMvcConfig;

    /**
     * Instantiates a new Soul client bean post processor.
     *
     * @param soulSpringMvcConfig the soul spring mvc config
     */
    public SpringMvcClientBeanPostProcessor(final SoulSpringMvcConfig soulSpringMvcConfig) {
        ValidateUtils.validate(soulSpringMvcConfig);
        // amdinUrl:http://localhost:9095
        // contextPath:/http
        // appName:http
        // full:false
        // host:null
        // port:8188
        this.soulSpringMvcConfig = soulSpringMvcConfig;
        url = soulSpringMvcConfig.getAdminUrl() + "/soul-client/springmvc-register";
        executorService = new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>());
    }

    @Override
    public Object postProcessAfterInitialization(@NonNull final Object bean, @NonNull final String beanName) throws BeansException {
        if (soulSpringMvcConfig.isFull()) {
            return bean;
        }
        // 判断是否有下面三个HTTP注解之一
        Controller controller = AnnotationUtils.findAnnotation(bean.getClass(), Controller.class);
        RestController restController = AnnotationUtils.findAnnotation(bean.getClass(), RestController.class);
        RequestMapping requestMapping = AnnotationUtils.findAnnotation(bean.getClass(), RequestMapping.class);
        if (controller != null || restController != null || requestMapping != null) {
            SoulSpringMvcClient clazzAnnotation = AnnotationUtils.findAnnotation(bean.getClass(), SoulSpringMvcClient.class);
            String prePath = "";
            if (Objects.nonNull(clazzAnnotation)) {
                // 类上有注解，且是通配符，以通配符方式进行注册,RPCType直接定死为HTTP
                if (clazzAnnotation.path().indexOf("*") > 1) {
                    String finalPrePath = prePath;
                    executorService.execute(() -> RegisterUtils.doRegister(buildJsonParams(clazzAnnotation, finalPrePath), url,
                            RpcTypeEnum.HTTP));
                    return bean;
                }
                prePath = clazzAnnotation.path();
            }
            // 遍历所有的方法，注册上面有Soul相关注解的方法
            final Method[] methods = ReflectionUtils.getUniqueDeclaredMethods(bean.getClass());
            for (Method method : methods) {
                SoulSpringMvcClient soulSpringMvcClient = AnnotationUtils.findAnnotation(method, SoulSpringMvcClient.class);
                if (Objects.nonNull(soulSpringMvcClient)) {
                    String finalPrePath = prePath;
                    executorService.execute(() -> RegisterUtils.doRegister(buildJsonParams(soulSpringMvcClient, finalPrePath), url,
                            RpcTypeEnum.HTTP));
                }
            }
        }
        return bean;
    }

    // 构造请求内容
    private String buildJsonParams(final SoulSpringMvcClient soulSpringMvcClient, final String prePath) {
        String contextPath = soulSpringMvcConfig.getContextPath();
        String appName = soulSpringMvcConfig.getAppName();
        Integer port = soulSpringMvcConfig.getPort();
        String path = contextPath + prePath + soulSpringMvcClient.path();
        String desc = soulSpringMvcClient.desc();
        String configHost = soulSpringMvcConfig.getHost();
        String host = StringUtils.isBlank(configHost) ? IpUtils.getHost() : configHost;
        String configRuleName = soulSpringMvcClient.ruleName();
        String ruleName = StringUtils.isBlank(configRuleName) ? path : configRuleName;
        SpringMvcRegisterDTO registerDTO = SpringMvcRegisterDTO.builder()
                .context(contextPath)
                .host(host)
                .port(port)
                .appName(appName)
                .path(path)
                .pathDesc(desc)
                .rpcType(soulSpringMvcClient.rpcType())
                .enabled(soulSpringMvcClient.enabled())
                .ruleName(ruleName)
                .registerMetaData(soulSpringMvcClient.registerMetaData())
                .build();
        return OkHttpTools.getInstance().getGson().toJson(registerDTO);
    }
}
```

### 注册配置信息来源
&ensp;&ensp;&ensp;&ensp;我们来大致看一下那些注册信息的来源

&ensp;&ensp;&ensp;&ensp;下面这个是一个需要注册的接口：

```java
@RestController
@RequestMapping("/order")
@SoulSpringMvcClient(path = "/order")
public class OrderController {

    @PostMapping("/save")
    @SoulSpringMvcClient(path = "/save" , desc = "Save order")
    public OrderDTO save(@RequestBody final OrderDTO orderDTO) {
        orderDTO.setName("hello world save order");
        return orderDTO;
    }
}
```

&ensp;&ensp;&ensp;&ensp;这个是soul相关的配置文件：

```xml
soul:
  http:
    adminUrl: http://localhost:9095
    port: 8188
    contextPath: /http
    appName: http
    full: false
```

&ensp;&ensp;&ensp;&ensp;下面这个是POST的内容，我们可以看到：AppName、context、port都是从配置中获取

&ensp;&ensp;&ensp;&ensp;path、pathDesc、RPCType等等都在在程序注解中获取的（SoulSpringMvcClient），enabled默认值是true

```json
{
	"appName": "http",
	"context": "/http",
	"path": "/http/order/save",
	"pathDesc": "Save order",
	"rpcType": "http",
	"host": "172.21.160.1",
	"port": 8188,
	"ruleName": "/http/order/save",
	"enabled": true,
	"registerMetaData": false
}
```

&ensp;&ensp;&ensp;&ensp;简单去看一下Soul-Admin的接口处理函数，可以看到是从传入的数据中进行选择器和规则的处理，和我们想象的差不多（这里深入下去代码太多了，就不详解，CRUD大家应该看起来没得问题),需要注意的是，在处理的过程中会触发事件发布，进一步验证了Controller是数据同步事件的触发入口

```java
public class SoulClientRegisterServiceImpl implements SoulClientRegisterService {
    @Override
    @Transactional
    public String registerSpringMvc(final SpringMvcRegisterDTO dto) {
        if (dto.isRegisterMetaData()) {
            MetaDataDO exist = metaDataMapper.findByPath(dto.getPath());
            if (Objects.isNull(exist)) {
                saveSpringMvcMetaData(dto);
            }
        }
        // 处理选择器
        String selectorId = handlerSpringMvcSelector(dto);
        // 处理规则
        handlerSpringMvcRule(selectorId, dto);
        return SoulResultMessage.SUCCESS;
    }
}
```

## 总结
&ensp;&ensp;&ensp;&ensp;本篇初步探索了下Soul-Client的HTTP注册注解，了解了注册的大致流程：

- 1.如果类上有Controller、RestController、RequestMapping这三个注解之一，那将进行下一个Soul注解的判断注册
- 2.如果是在类上有Soul的相关注解
  - 如果注解是通配符，将所有接口都进行注册
  - 如果不是，先得到接口前缀，获取所有方法，将方法上有Soul相关注解的都给注册上去 

&ensp;&ensp;&ensp;&ensp;注册的信息基本都是从配置文件和注解中进行获取的

&ensp;&ensp;&ensp;&ensp;注册的内容是相应的选择器和规则

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