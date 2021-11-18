# 数据库连接池Demo（2）多线程初步
***
## 简介
本文在之前文章Druid使用流程文章基础下，来实现在多线程请求下的数据库连接池初步Demo

## 探索Alibaba Druid的物理连接生成
参考Alibaba Druid数据库的实现：[Alibaba Druid 源码阅读（六）数据库连接使用流程初探](https://juejin.cn/post/7031409660552986632/)

我们主要的实现思路如下：

- 1.初始化配置的初始物理连接数
- 2.获取连接时，从空闲线程池中阻塞获取
- 3.获取连接时，发送生成物理连接指令去生成新的物理连接，但物理连接数不得大于配置的最大连接数
- 4.连接关闭时，归还空闲线程池

在Alibaba Druid中是使用等待-通知机制来阻塞获取的，我们简单点，就用一个阻塞队列实现这个

### 1.初始化生成配置初始化数据的物理连接
先简单直接的在构造函数中生成,并将其放入空闲池中：

```java
public class SelfDataSource implements DataSource {

    /**
     * 放置空闲可用的连接
     */
    private final LinkedBlockingDeque<SelfPoolConnection> idle = new LinkedBlockingDeque<>();
    /**
     * 放置正在使用的连接
     */
    private final LinkedBlockingDeque<SelfPoolConnection> active = new LinkedBlockingDeque<>();
    private final AtomicInteger connectCount = new AtomicInteger(0);

    private final String url;
    private final String username;
    private final String password;
    private int maxActive;
    private int timeout = 100;

    public SelfDataSource(final Properties properties) {
        this.url = properties.getProperty("url");
        this.username = properties.getProperty("username");
        this.password = properties.getProperty("password");
        this.maxActive = Integer.parseInt(properties.getProperty("maxActive"));
        this.timeout = Integer.parseInt(properties.getProperty("timeout", "100"));

        final int initCount = Integer.parseInt(properties.getProperty("initCount", "0"));
        if (initCount > maxActive) {
            throw new RuntimeException("initCount gt maxActive");
        }

        IntStream.range(0, initCount).forEach(i -> createPhysicsConnect());
    }

    private void createPhysicsConnect() {
        System.out.println("生成新物理连接");
        try {
            idle.put(new SelfPoolConnection(this, url, username, password));
            connectCount.addAndGet(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

### 2.从池中获取物理连接
获取连接时，从空闲池中进行获取，返回活跃使用池中：

```java
public class SelfDataSource implements DataSource {

    /**
     * 放置空闲可用的连接
     */
    private final LinkedBlockingDeque<SelfPoolConnection> idle = new LinkedBlockingDeque<>();
    /**
     * 放置正在使用的连接
     */
    private final LinkedBlockingDeque<SelfPoolConnection> active = new LinkedBlockingDeque<>();

    /**
     * 自定义的获取数据库物理连接
     * 1.无空闲连接则生成新的物理连接，并且放入正在运行连接池中
     * 2.如果有空闲连接，则获取，并放入正在运行连接池中
     * @return 自定义的数据库物理连接（自定义以能够自定义Close方法）
     * @throws SQLException
     */
    @Override
    public Connection getConnection() throws SQLException {
        try {
            System.out.println("获取连接，开始");
            final SelfPoolConnection connection = idle.take();
            active.put(connection);
            System.out.println("获取连接，结束\n");
            return connection;
        } catch (InterruptedException e) {
            e.printStackTrace();
            throw new RuntimeException(e);
        }
    }
}
```

### 3.回收连接
将连接从使用活跃池中移除，放入空闲池中：

```java
public class SelfDataSource implements DataSource {

    /**
     * 放置空闲可用的连接
     */
    private final LinkedBlockingDeque<SelfPoolConnection> idle = new LinkedBlockingDeque<>();
    /**
     * 放置正在使用的连接
     */
    private final LinkedBlockingDeque<SelfPoolConnection> active = new LinkedBlockingDeque<>();

    /**
     * 初步将Connection从运行池中异常，放入空闲池
     * 从正在使用连接池中移除，放入空闲连接池中
     * @param selfPoolConnection 自定义Connection
     */
    public void recycle(final SelfPoolConnection selfPoolConnection) {
        try {
            System.out.println("回收连接,开始");
            while (!active.remove(selfPoolConnection)) {}
            idle.put(selfPoolConnection);
            System.out.println("回收连接,结束\n");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```