# Alibaba Druid 源码阅读（一） 数据库连接池初步
***
这是我参与11月更文挑战的第N天，活动详情查看：2021最后一次更文挑战

## 简介
本文将初步探索数据库连接池的应用场景，为后面的源码分析做些准备

## 数据库连接池的应用场景
在没有连接池之前，在使用中，需要访问数据库时，需要如下步骤：

- 1.创建一个连接
- 2.使用
- 3.关闭连接

上面基本上是一个连接的使用生命周期，如果访问不是很频繁，那还行，但如果在频繁访问下就会有下面的问题：

- 1.耗时：一个连接的创建是比较耗时的，频繁的创建连接产生不必要的耗时
- 2.资源：创建和销毁都需要对应的服务销毁资源，频繁的创建连接浪费系统资源

为了解决上面的问题，可以通过连接复用：通过建立一个数据库连接池以及一套连接使用管理策略，使得一个数据库连接可以得到搞笑、安全的复用，避免数据库连接频繁建立、关闭的开销

对应设计模式：资源池。在日常的工作中，比如HTTP连接池、线程池等，也是应用资源池的思想

Druid数据库连接池，在国内基本上占统治地位，其代码还是值得研究的，所以后面就来研究一下其连接池的源码，提升下自己的能力

## Druid概览
其主要源码目录如下图：

其中重要的部分如下：

- filter：Druid支持监控、代理等功能，便于开发人员分析数据库操作的一些性能指标，主要就是使用filter实现的
- pool：连接池的主要实现
- sql：SQL语句解析
- wall：防火墙

### 简单的示例运行
下面我们就使用源码中的示例运行一下：druid-spring-boot-starter/src/test/java/com/alibaba/druid/spring/boot/demo/DemoApplication.java

```java
/**
 * @author lihengming<89921218@qq.com>
 *
 * <p>一个简单的使用演示</p>
 * <p>1.按需配置application.properties，配置项请参考config-template.properties</p>
 * <p>2.run DemoApplication</p>
 * <p>3.访问http://127.0.0.1:8080/druid</p>
 * <p>4.访问/user/${id}接口，查看SQL、Web、AOP监控效果，如：http://127.0.0.1:8080/user/1</p>
 *
 */
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

可以看到官方内置提供的快速体验的，基于内存数据H2的，我们访问下管理网页：http://127.0.0.1:8080/druid/sql.html

## 总结
由于之前这边研究不多，今天就简单热下身，为后面分析做些准备：

- 搜索相关的文章，了解了数据库连接池的应用场景
- 拉取Druid源码，运行示例

## 参考链接
- [Druid连接池介绍](https://github.com/alibaba/druid/wiki/Druid%E8%BF%9E%E6%8E%A5%E6%B1%A0%E4%BB%8B%E7%BB%8D)
- [数据库连接池的实现及原理](https://juejin.cn/post/6844903602939494414)
- [如何设计并实现一个db连接池？](https://juejin.cn/post/6844903853872119822)