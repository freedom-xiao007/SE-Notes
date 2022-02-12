# MyBatis3源码解析(8)MyBatis与Spring的结合
***

这是我参与2022首次更文挑战的第14天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7052884569032392740)

## 简介
在上几篇文章中，解析了MyBatis的核心原理部分，我们大致对其有了一定的了解，接下来我们看看在日常的开发中MyBatis是如何与Spring框架结合的

## 源码解析
在我们的日常开发中，使用Spring框架结合MyBatis，只需简单的进行相关的配置，在代码中使用注解即可轻松使用，而不用像下面示例中的，需要很多的代码：

```java
public class MybatisTest {

    // 从SqlSessionFactory中获取Mapper进行使用
    @Test
    public void test() {
        try(SqlSession session = buildSqlSessionFactory().openSession()) {
            PersonMapper personMapper = session.getMapper(PersonMapper.class);
            personMapper.createTable();
            personMapper.save(Person.builder().id(1L).name(new String[]{"1", "2"}).build());
            personMapper.save(Person.builder().id(2L).name(new String[]{"1", "2"}).build());

            Map<String, Object> query = new HashMap<>(2);
            query.put("id", 1);
            query.put("name", new String[]{"1", "2"});
            System.out.println(personMapper.getPersonByMap(query));
        }
    }

    // 初始化SqlSessionFactory
    public static SqlSessionFactory buildSqlSessionFactory() {
        String JDBC_DRIVER = "org.h2.Driver";
        String DB_URL = "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1";
        String USER = "sa";
        String PASS = "";
        DataSource dataSource = new PooledDataSource(JDBC_DRIVER, DB_URL, USER, PASS);
        Environment environment = new Environment("Development", new JdbcTransactionFactory(), dataSource);
        Configuration configuration = new Configuration(environment);
        configuration.getTypeHandlerRegistry().register(String[].class, JdbcType.VARCHAR, StringArrayTypeHandler.class);
        configuration.addMapper(PersonMapper.class);
        SqlSessionFactoryBuilder builder = new SqlSessionFactoryBuilder();
        return builder.build(configuration);
    }
}
```

在上面的代码中，我们可以明显的看到大段的SqlSessionFactory初始化和Mapper的获取的繁琐代码，但我们在Spring中没有这些繁琐的操作，如下变可以使用：

```java
@AllArgsConstructor
@Service
public class PersonService {

    private final PersonMapper personMapper;

    public void select() {
        personMapper.getPersonById(0);
    }
}
```

可以说非常省心和简单了

以前自己也不知道其原理，通过这段时间的学习，了解了MyBatis的原始使用和Spring的相关原理，知道了其背后的原理，在开发中使用起来感觉心中比以前有数了，希望通过这些源码解析的文章，也能对大家有所帮助

### SqlSessionFactory的初始化
这部分涉及到Spring的Bean和Spring Boot自动配置

可以尝试去自己写一个Spring Boot Starter，这样会有比较大帮助（以前好像写过一篇，但找不到了，可以参考下下面其他人的链接）:

- [SpringBoot自定义starter及自动配置](https://juejin.im/post/6844903988958232583)
- [spring boot 自定义配置属性的各种方式](https://blog.csdn.net/xujian_2001/article/details/79027026)

这里我们直接去看包：org.mybatis.spring.boot.autoconfigure

可以看到下面一段我们熟悉的内容：

可以看到Spring 在启动时便自动初始化了SqlSessionFactory,其中的细节这里就不赘述了

```java
    @Bean
    @ConditionalOnMissingBean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        factory.setDataSource(dataSource);
        factory.setVfs(SpringBootVFS.class);
        if (StringUtils.hasText(this.properties.getConfigLocation())) {
            factory.setConfigLocation(this.resourceLoader.getResource(this.properties.getConfigLocation()));
        }

        this.applyConfiguration(factory);
        if (this.properties.getConfigurationProperties() != null) {
            factory.setConfigurationProperties(this.properties.getConfigurationProperties());
        }

        if (!ObjectUtils.isEmpty(this.interceptors)) {
            factory.setPlugins(this.interceptors);
        }

        if (this.databaseIdProvider != null) {
            factory.setDatabaseIdProvider(this.databaseIdProvider);
        }

        if (StringUtils.hasLength(this.properties.getTypeAliasesPackage())) {
            factory.setTypeAliasesPackage(this.properties.getTypeAliasesPackage());
        }

        if (this.properties.getTypeAliasesSuperType() != null) {
            factory.setTypeAliasesSuperType(this.properties.getTypeAliasesSuperType());
        }

        if (StringUtils.hasLength(this.properties.getTypeHandlersPackage())) {
            factory.setTypeHandlersPackage(this.properties.getTypeHandlersPackage());
        }

        if (!ObjectUtils.isEmpty(this.typeHandlers)) {
            factory.setTypeHandlers(this.typeHandlers);
        }

        if (!ObjectUtils.isEmpty(this.properties.resolveMapperLocations())) {
            factory.setMapperLocations(this.properties.resolveMapperLocations());
        }

        Set<String> factoryPropertyNames = (Set)Stream.of((new BeanWrapperImpl(SqlSessionFactoryBean.class)).getPropertyDescriptors()).map(FeatureDescriptor::getName).collect(Collectors.toSet());
        Class<? extends LanguageDriver> defaultLanguageDriver = this.properties.getDefaultScriptingLanguageDriver();
        if (factoryPropertyNames.contains("scriptingLanguageDrivers") && !ObjectUtils.isEmpty(this.languageDrivers)) {
            factory.setScriptingLanguageDrivers(this.languageDrivers);
            if (defaultLanguageDriver == null && this.languageDrivers.length == 1) {
                defaultLanguageDriver = this.languageDrivers[0].getClass();
            }
        }

        if (factoryPropertyNames.contains("defaultScriptingLanguageDriver")) {
            factory.setDefaultScriptingLanguageDriver(defaultLanguageDriver);
        }

        this.applySqlSessionFactoryBeanCustomizers(factory);
        return factory.getObject();
    }
```

### Mapper的生成使用
这部分涉及到Spring的依赖注入相关的原理了，自己以前写过一个demo，但文章还没完善，后面抽空完善后，再发出来

简答来说，核心是：

- 1.应用启动时，自动生成需要的类实例放入容器中， 比如加了Service注解的类
- 2.对类实例进行初始化，在构造函数中，将其需要的类实例给传递进去

比如下面的：

```java
@AllArgsConstructor
@Service
public class PersonService {

    private final PersonMapper personMapper;
}
```

在Spring容器中已经有了类 PersonService 和 PersonMapper 的实例

在初始化阶段，调用 PersonService 构造函数时，发现需要 PersonMapper，由于从容器中取PersonMapper给其传递进去

大体的意思应该是这样，可能细节上会有些出入

下面是大致一些Debug的信息，通过调试就会发现，SqlSessionFactory 和 Mapper 初始化和获取等代码都是在应用程序其中就会执行触发

这个是PersonService这个bean初始化的相关栈函数：

```java
   @Nullable
    protected Object resolveAutowiredArgument(MethodParameter param, String beanName, @Nullable Set<String> autowiredBeanNames, TypeConverter typeConverter, boolean fallback) {
        Class<?> paramType = param.getParameterType();
        if (InjectionPoint.class.isAssignableFrom(paramType)) {
            InjectionPoint injectionPoint = (InjectionPoint)currentInjectionPoint.get();
            if (injectionPoint == null) {
                throw new IllegalStateException("No current InjectionPoint available for " + param);
            } else {
                return injectionPoint;
            }
        } else {
            try {
                return this.beanFactory.resolveDependency(new DependencyDescriptor(param, true), beanName, autowiredBeanNames, typeConverter);
            } catch (NoUniqueBeanDefinitionException var8) {
                throw var8;
            } catch (NoSuchBeanDefinitionException var9) {
                if (fallback) {
                    if (paramType.isArray()) {
                        return Array.newInstance(paramType.getComponentType(), 0);
                    }

                    if (CollectionFactory.isApproximableCollectionType(paramType)) {
                        return CollectionFactory.createCollection(paramType, 0);
                    }

                    if (CollectionFactory.isApproximableMapType(paramType)) {
                        return CollectionFactory.createMap(paramType, 0);
                    }
                }

                throw var9;
            }
        }
    }
```

调试的相关信息如下：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b8d709a4bee94795b0e2b8710815e3c9~tplv-k3u1fbpfcp-watermark.image?)

这个beanName是PersonService，需要的构造函数参数是 PersonMapper

一直跟着下面，它会从SqlSessionTemplate中取

```java
    public <T> T getMapper(Class<T> type) {
        return this.getConfiguration().getMapper(type, this);
    }
```

然后就跳到我们熟悉的 Mybatis类中 Configuration：

```java
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
```

## 总结
本篇中大致探索了：

- MyBatis初始化 SqlSessionFactory:Spring boot 自动配置
- Mapper初始化和使用：基于依赖注入，方便快捷的使用