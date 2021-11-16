# 数据库连接池Demo（2）多线程初步
***
## 简介
本文开始来探索下，多线程请求下的数据库连接池初步简单策略

## 探索Alibaba Druid的物理连接生成
### 1.在Alibaba Druid生成物理连接的函数中添加打印字段
如下：

```java
    # 类：DruidAbstractDataSource.java
    public Connection createPhysicalConnection(String url, Properties info) throws SQLException {
        Connection conn;
        if (getProxyFilters().size() == 0) {
            conn = getDriver().connect(url, info);
        } else {
            conn = new FilterChainImpl(this).connection_connect(info);
        }

        createCountUpdater.incrementAndGet(this);
        System.out.println("生成物理连接");

        return conn;
    }
```

然后运行我们下面的示例代码：

```java
package com.alibaba.druid;

import com.alibaba.druid.pool.DruidDataSource;

import java.sql.*;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;

public class TestDemo1 {


    public static final String DB_URL = "jdbc:h2:file:./demo-db";
    public static final String USER = "sa";
    public static final String PASS = "sa";
    public static final String QUERY = "SELECT id, name FROM user_example";


    public static void main(String[] args) throws InterruptedException, ExecutionException {
//        initData();

        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setInitialSize(0);
        dataSource.setMaxActive(3);
        dataSource.setMinIdle(2);
        dataSource.setDriverClassName("org.h2.Driver");
        dataSource.setUrl(DB_URL);
        dataSource.setUsername(USER);
        dataSource.setPassword(PASS);

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

        long cost = 0;
        for (int i = 0; i < 10; i++) {
            cost += (Long)(fs[i].get());
        }
        System.out.printf("一共花费：%d \n", cost);

        Thread.sleep(3000);
        fs = new FutureTask[5];
        for (int i=0; i<5; i++) {
            fs[i] = new FutureTask(() -> druidQuery(dataSource));
            new Thread(fs[i]).start();
        }

        Thread.sleep(3000);
        System.out.printf("当前数据库连接数：%d\n", dataSource.getActiveCount());
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

    private static long druidQuery(DruidDataSource dataSource) {
        System.out.println("开始执行查询");
        final long cur = System.currentTimeMillis();
        try(Connection conn = dataSource.getConnection();
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery(QUERY)) {
            // Extract data from result set
            while (rs.next()) {
                // Retrieve by column name
//                System.out.print("ID: " + rs.getInt("id"));
//                System.out.print(", name: " + rs.getString("name"));
//                System.out.print(";");
            }
//            System.out.println();
            System.out.printf("当前数据库连接数：%d\n", dataSource.getActiveCount());
        } catch (SQLException e) {
            e.printStackTrace();
        }
        final long cost = System.currentTimeMillis() - cur;
        System.out.println(cost);
        return cost;
    }
}
```

输出如下

```text
开始执行查询
开始执行查询
开始执行查询
开始执行查询
开始执行查询
开始执行查询
开始执行查询
开始执行查询
开始执行查询
开始执行查询
2021-11-16 20:51:01,047 [INFO ] ClickHouseDriver:42 - Driver registered
2021-11-16 20:51:01,063 [ERROR] DruidDataSource:1220 - testWhileIdle is true, validationQuery not set
createAndStartCreatorThread
2021-11-16 20:51:02,833 [INFO ] DruidDataSource:1017 - {dataSource-1} inited
生成物理连接
生成物理连接
生成物理连接
当前数据库连接数：3
当前数据库连接数：3
当前数据库连接数：3

当前数据库连接数：3

当前数据库连接数：3
当前数据库连接数：3

当前数据库连接数：3
当前数据库连接数：3
当前数据库连接数：2

当前数据库连接数：2

开始执行查询
开始执行查询
开始执行查询
当前数据库连接数：3
当前数据库连接数：3
开始执行查询
当前数据库连接数：3

开始执行查询

当前数据库连接数：2
当前数据库连接数：2

当前数据库连接数：0
```

可以看到虽然使用最大10个线程，但整个过程中，只触发示例三次生成物理连接的代码

### 2.定位如果触发生成物理连接
和之前自己文章中的不一样，这里并没有和自己预想的走到相关的函数，而是开启了一个线程去生成物理连接，并且使用了信号机制

```java
 	     createAndLogThread();
	    // 这去生成了那创建连接的线程
            createAndStartCreatorThread();
            System.out.println("createAndStartCreatorThread");
            createAndStartDestroyThread();

	    
    protected void createAndStartCreatorThread() {
        if (createScheduler == null) {
            String threadName = "Druid-ConnectionPool-Create-" + System.identityHashCode(this);
            createConnectionThread = new CreateConnectionThread(threadName);
            createConnectionThread.start();
            return;
        }

        initedLatch.countDown();
    }
```