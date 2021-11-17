# Alibaba Druid（6）数据库连接使用流程初探
***
这是我参与11月更文挑战的第6天，活动详情查看：2021最后一次更文挑战

## 简介
在写数据库Demo过程中，对如果当前连接最大数小于当前SQL查询线程时，是如何获取连接的，运行时的情况具体如何，下面就写相关的测试代码和debug相关的源码进行探索

## 一、线程多于连接最大数时的相关表现
先写一个测试相关代码：

- 初始化话相关数据
- 设置最大线程数为3，最下空闲数为2，初始化为0
- 起10个查询线程，等待所有结束
- 再起5个查询线程

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
        // 第一次运行可以初始化数据库数据，后面可以取消
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
            Thread.sleep(1000);
            System.out.printf("当前数据库连接数：%d\n", dataSource.getActiveCount());
        } catch (SQLException | InterruptedException e) {
            e.printStackTrace();
        }
        final long cost = System.currentTimeMillis() - cur;
        System.out.println(cost);
        return cost;
    }
}
```

我们来看下运行结果：

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
当前数据库连接数：3
当前数据库连接数：3
当前数据库连接数：3
当前数据库连接数：3
当前数据库连接数：3
当前数据库连接数：3
当前数据库连接数：3
当前数据库连接数：3
当前数据库连接数：3
当前数据库连接数：3

开始执行查询
开始执行查询
开始执行查询
开始执行查询
开始执行查询
当前数据库连接数：3
当前数据库连接数：3
当前数据库连接数：3
当前数据库连接数：3
当前数据库连接数：3
```

在上面的结果中，我们看到数据库连接数一直为3，下面就在源码中加上自己相关的日志，进行调试

## 二、源码中加上自定义日志
### 生成物理连接信息打印
在一起的分析文章中，我们看到了生成物理连接的相关代码，如下，在相关位置加上日志打印：

```java
# DruidAbstractDataSource.java
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

生成物理连接的最终代码，其是在一个线程中被调用的：DruidDataSource.java 里面的 CreateConnectionThread class

在上面的代码中加上日志后，我们再运行代码，发现这个函数只被调用了三次，确实最大连接数就是3

那连接数达到最大后，10个线程中剩下的是如何获取连接的呢？下面接着debug分析

### 连接数达到最大后，如何获取连接
回到getConnection的相关函数，我们调试发现，连接都是从 

```java

    private DruidPooledConnection getConnectionInternal(long maxWait) throws SQLException {
                // 下面的都是从connections里面去去连接
                // 但让人疑惑的是在createDirect和其相关的代码中，没有看到放入connection的相关操作
                // 后面再确认下这块的获取逻辑
                if (maxWait > 0) {
                    holder = pollLast(nanos);
                } else {
                    holder = takeLast();
		    System.out.println("get connection by takeLast");
                }
    }
```

如果加上相关的打印日志后，开始该方法一共被调用的15次

我们来详细查看下：

```java

    DruidConnectionHolder takeLast() throws InterruptedException, SQLException {
        try {
	    // 如果当前线程池里面没有可用连接，则会进行下面的循环，等待获取连接
            while (poolingCount == 0) {
                emptySignal(); // send signal to CreateThread create connection

                if (failFast && isFailContinuous()) {
                    throw new DataSourceNotAvailableException(createError);
                }

                notEmptyWaitThreadCount++;
                if (notEmptyWaitThreadCount > notEmptyWaitThreadPeak) {
                    notEmptyWaitThreadPeak = notEmptyWaitThreadCount;
                }
                try {
                    // 这个是关键，会进行线程登录，唤醒后，在后面从连接数组中获取连接
                    System.out.println("线程等待：" + System.currentTimeMillis());
                    notEmpty.await(); // signal by recycle or creator
                } finally {
                    System.out.println("线程唤醒：" + System.currentTimeMillis());
                    notEmptyWaitThreadCount--;
                }
                notEmptyWaitCount++;

                if (!enable) {
                    connectErrorCountUpdater.incrementAndGet(this);
                    if (disableException != null) {
                        throw disableException;
                    }

                    throw new DataSourceDisableException();
                }
            }
        } catch (InterruptedException ie) {
            notEmpty.signal(); // propagate to non-interrupted thread
            notEmptySignalCount++;
            throw ie;
        }

        // 获取连接
        decrementPoolingCount();
        DruidConnectionHolder last = connections[poolingCount];
        connections[poolingCount] = null;

        return last;
    }
```

在代码中，看到等待通知机制：

- 1.如果改方法被调用，线程池中有可用连接，则返回
- 2.没有则进行循环中，线程等待，唤醒后返回连接（超时机制之类，还没时间研究，后面再看）

看到获取线程是有两个条件的，记得我们在前几篇文中，获取连接，有个init函数，里面会先初始化配置的初始化连接数，如果配置了，那则跳过该循环

如果没有配置或者没有可用的，则会进行等待状态

那在哪些地方会唤醒该获取连接的函数呢，我们打上相关的日志后，接着往下看

###  如何唤醒获取连接的  takeLast

在前面查看生成物理连接时，我们发现了一处，如下，打上相关日志：

```java
    public class CreateConnectionThread extends Thread {
    ......
                        if (failFast) {
                            lock.lock();
                            try {
                                // 一处唤醒操作
                                notEmpty.signalAll();
                                System.out.println("生成物理连接，唤醒所有等待");
                            } finally {
                                lock.unlock();
                            }
    ......
         if (physicalConnection == null) {
                    continue;
                }

                System.out.println("生成连接后放入");
                boolean result = put(connection);
                if (!result) {
                    JdbcUtils.close(connection.getPhysicalConnection());
                    LOG.info("put physical connection to pool failed.");
                }
    }
```

经过日志加断点调试，发现并不是，于是查看了其他的函数，在put里面有相关的操作：

```java
private boolean put(DruidConnectionHolder holder, long createTaskId, boolean checkExists) {
    lock.lock();
    try {
        ......
        notEmpty.signal();
        System.out.println("连接获取，唤醒");
        notEmptySignalCount++;
        ......
    } finally {
        lock.unlock();
    }
    return true;
}
```

在上面的相关函数中打上日志

我们继续想想如果是我们自己使用的，那估计在连接close，归还连接到连接池的时候应该要通知一下，所有我们去看看recycle函数有没有相关的东西

果不其然，被我们找到了：在前面文章中，连接关闭时，会归还连接，放回连接池，具体函数如下

```java

    boolean putLast(DruidConnectionHolder e, long lastActiveTimeMillis) {
        if (poolingCount >= maxActive || e.discard || this.closed) {
            return false;
        }

        e.lastActiveTimeMillis = lastActiveTimeMillis;
        connections[poolingCount] = e;
        incrementPoolingCount();

        if (poolingCount > poolingPeak) {
            poolingPeak = poolingCount;
            poolingPeakTime = lastActiveTimeMillis;
        }

        notEmpty.signal();
        System.out.println("putLast，唤醒");
        notEmptySignalCount++;

        return true;
    }
```

顺手打上相关的日志

感觉差不多了，我们运行调式

### 运行结果

日志与分析如下：

```text
// 十个线程等待查询
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

// 物理连接生成线程启动
createAndStartCreatorThread

// 十个查询线程进入等待状态
get connection by takeLast start
线程等待：1637126886214
get connection by takeLast start
线程等待：1637126886214
get connection by takeLast start
线程等待：1637126886214
get connection by takeLast start
线程等待：1637126886214
get connection by takeLast start
线程等待：1637126886214
get connection by takeLast start
线程等待：1637126886214
get connection by takeLast start
线程等待：1637126886214
get connection by takeLast start
线程等待：1637126886214
get connection by takeLast start
线程等待：1637126886214
get connection by takeLast start
线程等待：1637126886215

// 下面是生成了三个物理连接，放入并唤醒
生成物理连接
生成连接后放入
连接获取，唤醒
线程唤醒：1637126886450
get connection by takeLast end

生成物理连接
生成连接后放入
连接获取，唤醒
线程唤醒：1637126886450
get connection by takeLast end

生成物理连接
生成连接后放入
连接获取，唤醒
线程唤醒：1637126886450
get connection by takeLast end

当前数据库连接数：3
当前数据库连接数：3

// 后面的都是连接回收后唤醒，然后获取连接执行查询
putLast，唤醒
线程唤醒：1637126887486
get connection by takeLast end
当前数据库连接数：3

putLast，唤醒
线程唤醒：1637126887486
get connection by takeLast end

putLast，唤醒
线程唤醒：1637126887488
get connection by takeLast end
当前数据库连接数：3
当前数据库连接数：3

putLast，唤醒
线程唤醒：1637126888488
get connection by takeLast end

putLast，唤醒
线程唤醒：1637126888488
get connection by takeLast end
当前数据库连接数：3

......

```

## 总结

本次编写了示例，并在源码中打印了相关的日志来验证阅读源码的所得，数据库连接池的连接基本使用流程如下：

1. 初始化，如果配置初始化连接数，则提前生成物理连接
2. 初始化时，启动物理连接生成线程（其最大生成数量不大于配置的最大数）
3. 获取连接，有则直接获取，没有则线程阻塞等待获取，等待唤醒后获取
   1. 物理连接生成，放入连接池中时，会唤醒
   2. 当连接归还线程时，会唤醒