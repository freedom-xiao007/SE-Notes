# Soul网关源码解析（二十七）Zookeeper注册中心
***
## 简介
本篇将详细介绍Zookeeper注册中心的的相关代码

## 概要
服务发现中有两个角色：Provider、Consumer

对应Soul网关里面就是Soul-Register-Center-Client、Soul-Register-Center-Server

register-client在Soul-Client中被调用，也就是需要的接入的后台服务中使用该模块

register-server在Soul-Admin中被调用，使用 Watch 等机制监听数据变化，进行相应的处理

下面来看看register-client是如何将需要接入的服务接口信息放入Zookeeper中和register-server如何监听获取数据处理

## Register-Client解析
### 示例运行
前面说过register-client在后台服务中使用，我们这里启动HTTP示例：soul-examples-http

修改配置注册方式为zookeeper，并填写zookeeper的服务地址：

```yaml
soul:
  client:
    registerType: zookeeper
    serverLists: localhost:2181
```

对了，确保自己运行了Zookeeper

### 源码跟踪
在soul-examples-http中，我们看到是使用的注解进行服务接口信息注册的，大致如下：SoulSpringMvcClient

```java
@RestController
@RequestMapping("/test")
@SoulSpringMvcClient(path = "/test/**")
public class HttpTestController {

    @PostMapping("/payment")
    public UserDTO post(@RequestBody final UserDTO userDTO) {
        return userDTO;
    }
}
```

注册的原理是实现Spring的BeanPostProcessor接口，对Bean进行自己的定制化操作，代码大致如下：

```java
@Slf4j
public class SpringMvcClientBeanPostProcessor implements BeanPostProcessor {

    // Distrupt publisher:用于将服务接口信息放入distrupt队列中
    private SoulClientRegisterEventPublisher publisher = SoulClientRegisterEventPublisher.getInstance();
    
    public SpringMvcClientBeanPostProcessor(final SoulRegisterCenterConfig config) {
        // 一推的获取配置信息的代码，省略
        ......
        // 这里使用SPI机制进行对应的注册中心的加载初始化和Distrupt的初始化
        // 可以参考:SPI全解析(https://blog.csdn.net/zm469568595/article/details/113362044)
        SoulClientRegisterRepository soulClientRegisterRepository = SoulClientRegisterRepositoryFactory.newInstance(config);
        publisher.start(soulClientRegisterRepository);
    }

    @Override
    public Object postProcessAfterInitialization(@NonNull final Object bean, @NonNull final String beanName) throws BeansException {
        if (isFull) {
            return bean;
        }
        Controller controller = AnnotationUtils.findAnnotation(bean.getClass(), Controller.class);
        RequestMapping requestMapping = AnnotationUtils.findAnnotation(bean.getClass(), RequestMapping.class);
        if (controller != null || requestMapping != null) {
            // 这里通过判断是否具有soul的特定注解：SoulSpringMvcClient，有的话进行服务注册
            // MVC有通配机制，这里就不详解了
            SoulSpringMvcClient clazzAnnotation = AnnotationUtils.findAnnotation(bean.getClass(), SoulSpringMvcClient.class);
            ......
            if (clazzAnnotation.path().indexOf("*") > 1) {
                String finalPrePath = prePath;
                // 这里将注册信息放入Distrupt中
                // Distrupt有点类似消息队列
                executorService.execute(() -> publisher.publishEvent(buildMetaDataDTO(clazzAnnotation, finalPrePath)));
                return bean;
            }
            ......
        }
        return bean;
    }
}
```

在上面的分析中看到了将服务信息放入Distrupt，下面来看看 SoulClientRegisterEventPublisher 的相关代码：

```java
public class SoulClientRegisterEventPublisher {

    public <T> void publishEvent(final T data) {
        DisruptorProvider<Object> provider = providerManage.getProvider();
        // 将数据放入队列中
        provider.onData(f -> f.setData(data));
    }
}
```

通过上面的代码，我们大致能猜到这是一个producer/consumer的模式，下面我们看看consumer相关的类代码：

```java
@SuppressWarnings("all")
public class RegisterClientConsumerExecutor extends QueueConsumerExecutor<DataTypeParent> {

    @Override
    public void run() {
        // 看到在这取出队列中的数据，并调用处理逻辑
        DataTypeParent result = getData();
        executorSubscriber.executor(Lists.newArrayList(result));
    }
}
```

跟着看看（进行调试即可跟进去）

```java
public class SoulClientMetadataExecutorSubscriber implements ExecutorTypeSubscriber<MetaDataRegisterDTO> {
    
    private final SoulClientRegisterRepository soulClientRegisterRepository;
    
    /**
     * Instantiates a new Soul client metadata executor subscriber.
     *
     * @param soulClientRegisterRepository the soul client register repository
     */
    public SoulClientMetadataExecutorSubscriber(final SoulClientRegisterRepository soulClientRegisterRepository) {
        this.soulClientRegisterRepository = soulClientRegisterRepository;
    }
    
    @Override
    public DataType getType() {
        return DataType.META_DATA;
    }
    
    @Override
    public void executor(final Collection<MetaDataRegisterDTO> metaDataRegisterDTOList) {
        // 在这里调用注册中心的相关处理逻辑对数据进行处理
        for (MetaDataRegisterDTO metaDataRegisterDTO : metaDataRegisterDTOList) {
            soulClientRegisterRepository.persistInterface(metaDataRegisterDTO);
        }
    }
}
```

我们再跟到persistInterface函数中，发现来到了： ZookeeperClientRegisterRepository

```java
public class ZookeeperClientRegisterRepository implements SoulClientRegisterRepository {

    @Override
    public void persistInterface(final MetaDataRegisterDTO metadata) {
        String rpcType = metadata.getRpcType();
        String contextPath = metadata.getContextPath().substring(1);
        // 将服务接口数据写到Zookeeper节点中
        registerMetadata(rpcType, contextPath, metadata);
        // 如果是http/tars/grpc，需要再写一份数据到uri节点下，用于进行服务的上下线
        if (RpcTypeEnum.HTTP.getName().equals(rpcType) || RpcTypeEnum.TARS.getName().equals(rpcType) || RpcTypeEnum.GRPC.getName().equals(rpcType)) {
            registerURI(rpcType, contextPath, metadata);
        }
        log.info("{} zookeeper client register success: {}", rpcType, metadata.toString());
    }
}
```

ZookeeperClientRegisterRepository 进行具体的Zookeeper信息写入，写入的数据结构如下：

```
soul
   ├──regsiter
   ├    ├──metadata
   ├    ├     ├──${rpcType}
   ├    ├     ├      ├────${contextPath}
   ├    ├     ├               ├──${ruleName} : save metadata data of MetaDataRegisterDTO
   ├    ├──uri
   ├    ├     ├──${rpcType}
   ├    ├     ├      ├────${contextPath}
   ├    ├     ├               ├──${ip:prot} : save uri data of URIRegisterDTO
   ├    ├     ├               ├──${ip:prot}
```

OK，到这Zookeeper相关的注册就写完了

## Register-Server解析
### Soul-Admin运行
Register-Server相关服务是随着Soul-Admin运行的

我们修改Soul-Admin的注册中心方式为Zookeeper，大致如下：

```yaml
soul:
  register:
    registerType: zookeeper
    serverLists : localhost:2181
```

### 源码解析
Register-Server的配置其中的文件如下：

```java
@Slf4j
@Configuration
public class RegisterCenterConfiguration {

    @Bean
    public SoulServerRegisterRepository soulServerRegisterRepository(final SoulRegisterCenterConfig soulRegisterCenterConfig, 
                                                                     final SoulClientRegisterService soulClientRegisterService) {
        String registerType = soulRegisterCenterConfig.getRegisterType();
        // 使用SPI进行相关注册中心的加载初始化
        SoulServerRegisterRepository registerRepository = ExtensionLoader.getExtensionLoader(SoulServerRegisterRepository.class).getJoin(registerType);
        RegisterServerDisruptorPublisher publisher = RegisterServerDisruptorPublisher.getInstance();
        publisher.start(soulClientRegisterService);
        registerRepository.init(publisher, soulRegisterCenterConfig);
        return registerRepository;
    }
}
```

跟着进去来到：

```java
@Slf4j
@Join
public class ZookeeperServerRegisterRepository implements SoulServerRegisterRepository {
    
    private static final EnumSet<RpcTypeEnum> METADATA_SET = EnumSet.of(RpcTypeEnum.DUBBO, RpcTypeEnum.GRPC, RpcTypeEnum.HTTP, RpcTypeEnum.SPRING_CLOUD, RpcTypeEnum.SOFA, RpcTypeEnum.TARS);
    
    private static final EnumSet<RpcTypeEnum> URI_SET = EnumSet.of(RpcTypeEnum.GRPC, RpcTypeEnum.HTTP, RpcTypeEnum.TARS);
    
    private void initSubscribe() {
        // 需要对Metadata和Uri进行监听
        // metadata 是具体服务注册信息，如selector和rules
        // uri 对应upstream，目的用于探活或者服务的上下线
        METADATA_SET.forEach(rpcTypeEnum -> subscribeMetaData(rpcTypeEnum.getName()));
        URI_SET.forEach(rpcTypeEnum -> subscribeURI(rpcTypeEnum.getName()));
    }
    
    // 监听对应rpctype的uri目录
    private void subscribeURI(final String rpcType) {
        String contextPathParent = RegisterPathConstants.buildURIContextPathParent(rpcType);
        List<String> contextPaths = zkClientGetChildren(contextPathParent);
        for (String contextPath : contextPaths) {
            watcherURI(rpcType, contextPath);
        }
        zkClient.subscribeChildChanges(contextPathParent, (parentPath, currentChildren) -> {
            if (CollectionUtils.isNotEmpty(currentChildren)) {
                for (String contextPath : currentChildren) {
                    watcherURI(rpcType, contextPath);
                }
            }
        });
    }

   // 监听对应rpctype的metadata目录
    private void subscribeMetaData(final String rpcType) {
        String contextPathParent = RegisterPathConstants.buildMetaDataContextPathParent(rpcType);
        List<String> contextPaths = zkClientGetChildren(contextPathParent);
        for (String contextPath : contextPaths) {
            watcherMetadata(rpcType, contextPath);
        }
        zkClient.subscribeChildChanges(contextPathParent, (parentPath, currentChildren) -> {
            if (CollectionUtils.isNotEmpty(currentChildren)) {
                for (String contextPath : currentChildren) {
                    watcherMetadata(rpcType, contextPath);
                }
            }
        });
    }
    
    // 读取数据进行服务注册处理，并监听该目录，有数据变动时进行服务注册处理
    private void watcherMetadata(final String rpcType, final String contextPath) {
        String metaDataParentPath = RegisterPathConstants.buildMetaDataParentPath(rpcType, contextPath);
        List<String> childrenList = zkClientGetChildren(metaDataParentPath);
        if (CollectionUtils.isNotEmpty(childrenList)) {
            childrenList.forEach(children -> {
                String realPath = RegisterPathConstants.buildRealNode(metaDataParentPath, children);
                publishMetadata(zkClient.readData(realPath).toString());
                subscribeMetaDataChanges(realPath);
            });
        }
        zkClient.subscribeChildChanges(metaDataParentPath, (parentPath, currentChildren) -> {
            if (CollectionUtils.isNotEmpty(currentChildren)) {
                List<String> addSubscribePath = addSubscribePath(childrenList, currentChildren);
                addSubscribePath.stream().map(addPath -> {
                    String realPath = RegisterPathConstants.buildRealNode(parentPath, addPath);
                    publishMetadata(zkClient.readData(realPath).toString());
                    return realPath;
                }).forEach(this::subscribeMetaDataChanges);
            
            }
        });
    }
    
    private void watcherURI(final String rpcType, final String contextPath) {
        String uriParentPath = RegisterPathConstants.buildURIParentPath(rpcType, contextPath);
        List<String> childrenList = zkClientGetChildren(uriParentPath);
        if (CollectionUtils.isNotEmpty(childrenList)) {
            registerURIChildrenList(childrenList, uriParentPath);
        }
        zkClient.subscribeChildChanges(uriParentPath, (parentPath, currentChildren) -> {
            if (CollectionUtils.isNotEmpty(currentChildren)) {
                registerURIChildrenList(currentChildren, parentPath);
            } else {
                registerURIChildrenList(new ArrayList<>(), parentPath);
            }
        });
    }
    
    // 这里uri处理需要注意：传入的是当前的存在uri列表，如果为空需要传入构造一个空的uri数据
    private void registerURIChildrenList(final List<String> childrenList, final String uriParentPath) {
        List<URIRegisterDTO> registerDTOList = new ArrayList<>();
        childrenList.forEach(addPath -> {
            String realPath = RegisterPathConstants.buildRealNode(uriParentPath, addPath);
            registerDTOList.add(GsonUtils.getInstance().fromJson(zkClient.readData(realPath).toString(), URIRegisterDTO.class));
        });
        if (registerDTOList.isEmpty()) {
            String contextPath = StringUtils.substringAfterLast(uriParentPath, "/");
            URIRegisterDTO uriRegisterDTO = URIRegisterDTO.builder().contextPath("/" + contextPath).build();
            registerDTOList.add(uriRegisterDTO);
        }
        publishRegisterURI(registerDTOList);
    }
    
    private void subscribeMetaDataChanges(final String realPath) {
        zkClient.subscribeDataChanges(realPath, new IZkDataListener() {
            @Override
            public void handleDataChange(final String dataPath, final Object data) {
                publishMetadata(data.toString());
            }
        
            @SneakyThrows
            @Override
            public void handleDataDeleted(final String dataPath) {
              
            }
        });
    }
    
    // 将metadata注册信息放入distrupt队列中
    private void publishMetadata(final String data) {
        publisher.publish(Lists.newArrayList(GsonUtils.getInstance().fromJson(data, MetaDataRegisterDTO.class)));
    }
    
    // 将uri注册信息放入distrupt队列中
    private void publishRegisterURI(final List<URIRegisterDTO> registerDTOList) {
        publisher.publish(registerDTOList);
    }
}
```

从上面的代码中，我们看到了Zookeeper如何取监听和取数据进行处理

下面我们跟着看看，数据如何进行处理的，相关的distrupt代码类似Soul-Client那边的：

```java
public class RegisterServerDisruptorPublisher implements SoulServerRegisterPublisher {
    
    @Override
    public <T> void publish(final T data) {
        DisruptorProvider<Object> provider = providerManage.getProvider();
        // 数据放入队列
        provider.onData(f -> f.setData(data));
    }
}
```

```java
public final class RegisterServerConsumerExecutor extends QueueConsumerExecutor<List<DataTypeParent>> {

    @Override
    public void run() {
        // 获取数据，根据是metadata或者uri调用不同的处理逻辑
        List<DataTypeParent> results = getData();
        getType(results).executor(results);
    }
    
    private ExecutorSubscriber getType(final List<DataTypeParent> list) {
        if (list == null || list.isEmpty()) {
            return null;
        }
        DataTypeParent result = list.get(0);
        return subscribers.get(result.getType());
    }
}
```

metadata处理逻辑如下，大致是调用相应的注册服务即可：

```java
public class MetadataExecutorSubscriber implements ExecutorTypeSubscriber<MetaDataRegisterDTO> {
    
    @Override
    public void executor(final Collection<MetaDataRegisterDTO> metaDataRegisterDTOList) {
        for (MetaDataRegisterDTO metaDataRegisterDTO : metaDataRegisterDTOList) {
            if (metaDataRegisterDTO.getRpcType().equals(RpcTypeEnum.DUBBO.getName())) {
                soulClientRegisterService.registerDubbo(metaDataRegisterDTO);
            } else if (metaDataRegisterDTO.getRpcType().equals(RpcTypeEnum.SOFA.getName())) {
                soulClientRegisterService.registerSofa(metaDataRegisterDTO);
            } else if (metaDataRegisterDTO.getRpcType().equals(RpcTypeEnum.TARS.getName())) {
                soulClientRegisterService.registerTars(metaDataRegisterDTO);
            } else if (metaDataRegisterDTO.getRpcType().equals(RpcTypeEnum.HTTP.getName())) {
                soulClientRegisterService.registerSpringMvc(metaDataRegisterDTO);
            } else if (metaDataRegisterDTO.getRpcType().equals(RpcTypeEnum.SPRING_CLOUD.getName())) {
                soulClientRegisterService.registerSpringCloud(metaDataRegisterDTO);
            } else if (metaDataRegisterDTO.getRpcType().equals(RpcTypeEnum.GRPC.getName())) {
                soulClientRegisterService.registerGrpc(metaDataRegisterDTO);
            }
        }
    }
}
```

uri处理逻辑，根据数据构造好相应的upstream列表数据，调用相应的处理逻辑即可：

```java
public class URIRegisterExecutorSubscriber implements ExecutorTypeSubscriber<URIRegisterDTO> {
    
    @Override
    public void executor(final Collection<URIRegisterDTO> dataList) {
        Map<String, List<URIRegisterDTO>> listMap = dataList.stream().collect(Collectors.groupingBy(URIRegisterDTO::getContextPath));
        listMap.forEach((contextPath, dtoList) -> {
            List<String> uriList = new ArrayList<>();
            dataList.forEach(uriRegisterDTO -> {
                if (uriRegisterDTO.getHost() != null && uriRegisterDTO.getPort() != null) {
                    uriList.add(String.join(":", uriRegisterDTO.getHost(), uriRegisterDTO.getPort().toString()));
                }
            });
            soulClientRegisterService.registerURI(contextPath, uriList);
        });
    }
}
```

下面的就是Service层面的操作，更新数据库，发布数据同步事件，这里就不详细说了，前面的文章中有相关的

## 总结
下面我们来总结下注册流程：

- Register-Client
  - 1.加上相关的Soul注解，启动时将注册信息写入Distrupt队列
  - 2.Distrupt Consumer取数据，调用相关的注册中心服务进行数据注册（本篇中是Zookeeper）
  - 3.调用Zookeeper相关注册逻辑，将服务接口信息metadata和uridata写入Zookeeper中
- Register-Sever
  - 1.Zookeeper注册中心监听并取相关注册数据，将其放入Distrupt队列中
  - 2.Distrupt Consumer取数据，调用相关的metadata或者uridata的处理逻辑
  - 3.调用相关的Service注册逻辑，更新数据库，发布数据同步事件

详细的流程图可以参考官网：https://dromara.org/zh/projects/soul/register-center-design/

如果需要写一个其他类型的注册中心，因为使用SPI加载的原因，需要在下面的模块中的resources/META-INF/soul下的文件中填写新增的注册中心类型

- soul-register-client-xxxx
- soul-register-server-xxxx
- soul-admin
- soul-client-apache-dubbo(如果新增的client的注册中心刚开始不生效，可以在这加一下)

## Soul网关源码解析文章列表
&ensp;&ensp;&ensp;&ensp;对用Java写的高性能网关：[Soul](https://github.com/dromara/soul),进行一波学习和研究，下面是相关的文章记录

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