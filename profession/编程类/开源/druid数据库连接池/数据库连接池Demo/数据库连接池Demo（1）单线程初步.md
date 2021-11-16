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

单连接查询：

```java
/**
     * 原生JDBC查询 单连接查询
     */
    private static void rawSingleExample() {
        try(Connection conn = DriverManager.getConnection(DB_URL, USER, PASS)) {
            for (int i=0; i<queryAmount; i++) {
                try (Statement stmt = conn.createStatement();
                     ResultSet rs = stmt.executeQuery(QUERY);) {
                    while (rs.next()) {
                        System.out.print("ID: " + rs.getInt("id"));
                        System.out.print(", name: " + rs.getString("name"));
                        System.out.print(";");
                    }
                    System.out.println();
                }
            }
        } catch (SQLException e) {
            e.printStackTrace();
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
因为在测试代码中，涉及的需要实现的函数如下,其他的默认实现即可

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

#### 测试运行
测试函数如下：

```java
public class Main {

    /**
     * 自定义数据库连接池查询
     */
    private static void selfExample() {
        final SelfDataSource dataSource = new SelfDataSource(DB_URL, USER, PASS);
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

### 运行对比
运行函数与结果如下：

```java
public class Main {

    /**
     * 单线程测试代码
     * 注意：不要一起运行测试，感觉有缓存，导致在后面运行的查询速度很快
     *      需要运行那个单独放开，其他注释掉
     * @param args args
     * @throws Exception e
     */
    public static void main(String[] args) throws Exception {
//        initData();

        final StringBuilder result = new StringBuilder();
        long current = System.currentTimeMillis();

//        rawExample();
//        result.append(String.format("原生查询耗时：%d 毫秒\n", System.currentTimeMillis() - current));

//        rawSingleExample();
//        result.append(String.format("原生Jdbc单连接查询耗时：%d 毫秒\n", System.currentTimeMillis() - current));

//        druidExample();
//        result.append(String.format("Druid连接池查询耗时：%d 毫秒\n", System.currentTimeMillis() - current));

        selfExample();
        result.append(String.format("自写连接池查询耗时：%d 毫秒\n", System.currentTimeMillis() - current));

        System.out.println(result);
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
原生查询耗时：1715 毫秒

ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
原生Jdbc单连接查询耗时：770 毫秒

ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
ID: 1, name: 0;ID: 2, name: 1;ID: 3, name: 2;ID: 4, name: 3;ID: 5, name: 4;ID: 6, name: 5;ID: 7, name: 6;ID: 8, name: 7;ID: 9, name: 8;ID: 10, name: 9;
Druid连接池查询耗时：588 毫秒

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
自写连接池查询耗时：473 毫秒
```

## 总结
从接口上看，一部分是符合我们预期的：连接池的性能是远远优于不使用连接池的

但自定义的连接池竟然比Druid还快，是我没有想到的，一度有些怀疑

但相关的确实是实现了的，从日志上来看，确实是只初始化了一次，后面没有再初始物理连接

目前的例子是单线程，没有考虑加锁、检查、异常处理等，可能是这些有影响，后面我们再研究研究

代码参考地址，可查看Tag V0.0.1：https://github.com/lw1243925457/DataSourcePoolDemo

## 参考链接
- [Alibaba Druid 源码阅读（一） 数据库连接池初步](https://juejin.cn/post/7028200379338735623/)
- [Alibaba Druid 源码阅读（二） 数据库连接池实现初步探索 ](https://juejin.cn/post/7028580356353687566/)
- [Alibaba Druid 源码阅读（三） 数据库连接池初始化探索](https://juejin.cn/post/7028919903834865671/)
- [Alibaba Druid 源码阅读（四） 数据库连接池中连接获取探索](https://juejin.cn/post/7029288150258171935)
- [Alibaba Druid 源码阅读（五）数据库连接池 连接关闭探索](https://juejin.cn/post/7029666671870623781)