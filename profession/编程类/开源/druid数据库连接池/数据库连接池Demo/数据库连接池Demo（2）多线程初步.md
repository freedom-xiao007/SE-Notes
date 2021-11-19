# 数据库连接池Demo（2）多线程初步
***
这是我参与11月更文挑战的第7天，活动详情查看：2021最后一次更文挑战

## 简介

本文在之前文章Druid使用流程文章基础下，来实现在多线程请求下的数据库连接池初步Demo，实现一个多线程下能跑的最基础版本

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
            final SelfPoolConnection connection = idle.take();
            System.out.println("获取连接，结束\n");
            active.put(connection);
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
            while (!active.remove(selfPoolConnection)) {}
            System.out.println("回收连接,结束\n");
            idle.put(selfPoolConnection);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```



### 4.测试运行

测试的代码如下：

```java
public class MultiThreadSelfTest {

    public static final String DB_URL = "jdbc:h2:file:./demo-db";
    public static final String USER = "sa";
    public static final String PASS = "sa";
    public static final String QUERY = "SELECT id, name FROM user_example";


    public static void main(String[] args) throws InterruptedException, ExecutionException {
        // 第一次运行可以初始化数据库数据，后面可以取消
//        initData();

        final Properties properties = new Properties();
        properties.setProperty("url", "jdbc:h2:file:./demo-db");
        properties.setProperty("username", "sa");
        properties.setProperty("password", "sa");
        properties.setProperty("maxActive", "5");
        properties.setProperty("initCount", "5");

        SelfDataSource dataSource = new SelfDataSource(properties);

        FutureTask[] fs = new FutureTask[10];
        for (int i=0; i<10; i++) {
            fs[i] = new FutureTask(() -> druidQuery(dataSource));
            new Thread(fs[i]).start();
        }

        while (true) {
            for (int i = 0; i < 10; i++) {
                if (!fs[i].isDone()) {
                    continue;
                }
            }
            break;
        }
        
        Thread.sleep(3000);
        System.out.printf("当前数据库连接数：%d\n", dataSource.getConnectionCount());
    }

    /**
     * 生成数据
     */
    public static void initData() {
        final String drop = "drop table `user_example` if exists;";
        final String createTable = "CREATE TABLE IF NOT EXISTS `user_example` (" +
                "`id` bigint NOT NULL AUTO_INCREMENT, " +
                "`name` varchar(100) NOT NULL" +
                ");";
        final String addUser = "insert into user_example (name) values(%s)";
        try(Connection conn = DriverManager.getConnection(DB_URL, USER, PASS);
            Statement stmt = conn.createStatement()) {
            stmt.execute(drop);
            stmt.execute(createTable);
            for (int i=0; i<10; i++) {
                stmt.execute(String.format(addUser, i));
            }
            conn.commit();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    private static long druidQuery(SelfDataSource dataSource) {
        System.out.println("开始执行查询");
        final long cur = System.currentTimeMillis();
        try(Connection conn = dataSource.getConnection();
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery(QUERY)) {
            while (rs.next()) {
            }
            Thread.sleep(1000);
        } catch (SQLException | InterruptedException e) {
            e.printStackTrace();
        }
        final long cost = System.currentTimeMillis() - cur;
        System.out.println("查询结束，耗时：" + cost);
        return cost;
    }
}
```



结果及分析如下：

```tex
// 生成最初的五个物理连接
初始化物理连接
初始化物理连接
初始化物理连接
初始化物理连接
初始化物理连接

开始执行查询
开始执行查询
获取连接，结束 // 使用数 1 空闲数 4
开始执行查询
获取连接，结束 // 使用数 2 空闲数 3
开始执行查询
获取连接，结束 // 使用数 3 空闲数 2
开始执行查询
开始执行查询
获取连接，结束 // 使用数 4 空闲数 1
开始执行查询
获取连接，结束 // 使用数 5 空闲数 0
开始执行查询
开始执行查询
开始执行查询
回收连接,结束 // 使用数 4 空闲数 1
回收连接,结束 // 使用数 3 空闲数 2
回收连接,结束 // 使用数 2 空闲数 3

查询结束，耗时：1034
查询结束，耗时：1034
查询结束，耗时：1034

获取连接，结束 // 使用数 3 空闲数 2
回收连接,结束 // 使用数 2 空闲数 3
获取连接，结束 // 使用数 3 空闲数 2
回收连接,结束 // 使用数 2 空闲数 3

查询结束，耗时：1034

获取连接，结束 // 使用数 3 空闲数 2
获取连接，结束 // 使用数 4 空闲数 1
获取连接，结束 // 使用数 5 空闲数 0

查询结束，耗时：1035

回收连接,结束
回收连接,结束

查询结束，耗时：2041

回收连接,结束
回收连接,结束
回收连接,结束

查询结束，耗时：2042
查询结束，耗时：2041
查询结束，耗时：2041
查询结束，耗时：2042

当前数据库连接数：5

```



## 总结

运行结果看起来大致符合我们的预期，但总感觉还有有一些不对劲的地方，现在暂时还看不出来

下面还有一些未完成的：

- 最小空闲线程数的实现
- 动态线程的生成

后面继续完善吧，还有就是和Alibaba Druid的运行对比，这些后面继续写和研究

代码参考地址，可查看Tag V0.0.1：https://github.com/lw1243925457/DataSourcePoolDemo

## 参考链接
- [Alibaba Druid 源码阅读（一） 数据库连接池初步](https://juejin.cn/post/7028200379338735623/)
- [Alibaba Druid 源码阅读（二） 数据库连接池实现初步探索 ](https://juejin.cn/post/7028580356353687566/)
- [Alibaba Druid 源码阅读（三） 数据库连接池初始化探索](https://juejin.cn/post/7028919903834865671/)
- [Alibaba Druid 源码阅读（四） 数据库连接池中连接获取探索](https://juejin.cn/post/7029288150258171935)
- [Alibaba Druid 源码阅读（五）数据库连接池 连接关闭探索](https://juejin.cn/post/7029666671870623781)
- [Alibaba Druid 源码阅读（六）数据库连接使用流程初探](https://juejin.cn/post/7031409660552986632/)

- [数据库连接池Demo（1）单线程初步](https://juejin.cn/post/7030807425905066014/)