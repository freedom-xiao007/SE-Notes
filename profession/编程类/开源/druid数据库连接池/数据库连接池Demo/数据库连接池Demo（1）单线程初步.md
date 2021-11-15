# 数据库连接池Demo（1）单线程初步
***
这是我参与11月更文挑战的第6天，活动详情查看：2021最后一次更文挑战

## 简介
在上周阅读Alibaba Druid数据库连接池后，感觉光看有点领会不到精髓，后面这几篇文章将尝试自己实现一个数据库连接池Demo

## 原生JDBC与Alibaba Druid使用
我们先把相关的测试给搭建起来，把JDBC、Druid的相关示例代码跑起来，看看效果和性能

在实现一个最简单的自定义连接池，然后运行三者进行对比

### 原生JDBC
我们先使用原生JDBC将数据库数据初始化，然后去查询，代码如下：

```java
public class Main {

    static final String DB_URL = "jdbc:h2:file:./demo-db";
    static final String USER = "sa";
    static final String PASS = "sa";

    /**
     * 生成数据
     */
    private static void initData() {
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
}
```

然后原始暴力查询：

```java
public class Main {

    /**
     * 原生JDBC查询
    */
    private static void rawExample() {
        for (int i=0; i<queryAmount; i++) {
            // Open a connection
            try(Connection conn = DriverManager.getConnection(DB_URL, USER, PASS);
                Statement stmt = conn.createStatement();
                ResultSet rs = stmt.executeQuery(QUERY);) {
                // Extract data from result set
                while (rs.next()) {
                    // Retrieve by column name
                    System.out.print("ID: " + rs.getInt("id"));
                    System.out.print(", name: " + rs.getString("name"));
                    System.out.print(";");
                }
                System.out.println();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }
}
```

### Alibaba Druid查询
使用Alibaba Druid进行相同的查询操作：

```java
public class Main {
    /**
     * Alibaba Druid查询
    */
    private static void druidExample() throws Exception {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setInitialSize(1);
        dataSource.setMaxActive(1);
        dataSource.setMinIdle(1);
        dataSource.setDriverClassName("org.h2.Driver");
        dataSource.setUrl(DB_URL);
        dataSource.setUsername(USER);
        dataSource.setPassword(PASS);

        for (int i=0; i<queryAmount; i++) {
            // Open a connection
            try(Connection conn = dataSource.getConnection();
                Statement stmt = conn.createStatement();
                ResultSet rs = stmt.executeQuery(QUERY)) {
                // Extract data from result set
                while (rs.next()) {
                    // Retrieve by column name
                    System.out.print("ID: " + rs.getInt("id"));
                    System.out.print(", name: " + rs.getString("name"));
                    System.out.print(";");
                }
                System.out.println();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }
}
```

### 自写最简单版本查询
相关的思路如下：

- 1.基于接口Database写一个自己的Database：SelfDatasource
  - 实现自己的getConnection方法
- 2.基于接口：javax.sql.PooledConnection, Connection，自定义数据物理连接
  - 为了自定义close方法，将数据库连接回收复用

#### 自定义Database
列出来的就是要进行自己实现，其他方法暂时默认：

在测试代码中，就涉及到一个getConnection函数，以及自定义的连接池复用相关

```java
public class SelfDataSource implements DataSource {
    /**
     * 放置空闲可用的连接
     */
    private final Queue<SelfPoolConnection> idle = new LinkedList<>();
    /**
     * 放置正在使用的连接
     */
    private final Set<SelfPoolConnection> running = new HashSet<>();
    private final String url;
    private final String username;
    private final String password;

    public SelfDataSource(final String url, final String username, final String password) {
        this.url = url;
        this.username = username;
        this.password = password;
    }

    /**
     * 初步将Connection从运行池中异常，放入空闲池
     * 从正在使用连接池中移除，放入空闲连接池中
     * @param selfPoolConnection 自定义Connection
     */
    public void recycle(final SelfPoolConnection selfPoolConnection) {
        running.remove(selfPoolConnection);
        idle.add(selfPoolConnection);
        System.out.println("回收连接");
    }

    /**
     * 自定义的获取数据库物理连接
     * 1.无空闲连接则生成新的物理连接，并且放入正在使用连接池中
     * 2.如果有空闲连接，则获取，并放入正在使用连接池中
     * @return 自定义的数据库物理连接（自定义以能够自定义Close方法）
     * @throws SQLException
     */
    @Override
    synchronized public Connection getConnection() throws SQLException {
        if (idle.isEmpty()) {
            System.out.println("生成新物理连接");
            SelfPoolConnection conn = new SelfPoolConnection(this, url, username, password);
            running.add(conn);
            return conn.getConnection();
        }
        SelfPoolConnection conn = idle.poll();
        running.add(conn);
        return conn.getConnection();
    }
}
```

#### 自定义Connection
因为在测试代码中，涉及的需要实现的函数如下：

- createStatement
- close（try自动调用）
- getConnection（自定义DataSource调用）

```java
public class SelfPoolConnection implements javax.sql.PooledConnection, Connection {

    private final SelfDataSource selfDatasource;
    private Connection connection;

    public SelfPoolConnection(final SelfDataSource selfDatasource, final String url, final String username, final String password) {
        this.selfDatasource = selfDatasource;
        System.out.println("初始化物理连接");
        try {
            connection = DriverManager.getConnection(url, username, password);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    @Override
    public Statement createStatement() throws SQLException {
        return connection.createStatement();
    }

    @Override
    synchronized public Connection getConnection() throws SQLException {
        return this;
    }

    /**
     * 关闭连接，调用自定义DataSource用于复用
     * 目前感觉这样不规范，但时间紧张，前期先简单实现
     * @throws SQLException
     */
    @Override
    public void close() throws SQLException {
        selfDatasource.recycle(this);
    }
}
```

### 运行对比
运行函数与结果如下：

```java
public class Main {

    public static void main(String[] args) throws Exception {
        initData();

        long current = System.currentTimeMillis();
        rawExample();
        System.out.printf("原生查询耗时：%d 毫秒\n", System.currentTimeMillis() - current);

        current = System.currentTimeMillis();
        druidExample();
        System.out.printf("连接池查询耗时：%d 毫秒\n", System.currentTimeMillis() - current);

        current = System.currentTimeMillis();
        selfExample();
        System.out.printf("自写连接池查询耗时：%d 毫秒\n", System.currentTimeMillis() - current);

        Thread.sleep(3000);
    }
}
```

结果：

```
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
原生查询耗时：220 毫秒
11月 15, 2021 10:09:04 下午 com.alibaba.druid.pool.DruidDataSource error
严重: testWhileIdle is true, validationQuery not set
11月 15, 2021 10:09:04 下午 com.alibaba.druid.pool.DruidDataSource info
信息: {dataSource-1} inited
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
连接池查询耗时：69 毫秒
生成新物理连接
初始化物理连接
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
回收连接
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
回收连接
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
回收连接
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
回收连接
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
回收连接
自写连接池查询耗时：6 毫秒
Disconnected from the target VM, address: '127.0.0.1:54211', transport: 'socket'
```

## 总结
从接口上看，一部分是符合我们预期的：连接池的性能是远远优于不使用连接池的

但自定义的连接池竟然比Druid还快，是我没有想到的，一度有些怀疑

但相关的地址确实是实现了的，从日志上来看，确实是只初始化了一次，后面没有再初始物理连接

目前的例子是单线程，没有考虑加锁、检查、异常处理等，可能是这些有影响，后面我们再研究研究

代码参考地址：https://github.com/lw1243925457/DataSourcePoolDemo

## 参考链接
- [Alibaba Druid 源码阅读（一） 数据库连接池初步](https://juejin.cn/post/7028200379338735623/)
- [Alibaba Druid 源码阅读（二） 数据库连接池实现初步探索 ](https://juejin.cn/post/7028580356353687566/)
- [Alibaba Druid 源码阅读（三） 数据库连接池初始化探索](https://juejin.cn/post/7028919903834865671/)
- [Alibaba Druid 源码阅读（四） 数据库连接池中连接获取探索](https://juejin.cn/post/7029288150258171935)
- [Alibaba Druid 源码阅读（五）数据库连接池 连接关闭探索](https://juejin.cn/post/7029666671870623781)