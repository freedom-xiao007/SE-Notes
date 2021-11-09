# Alibaba Druid 源码阅读（二） 数据库连接池实现初步探索
***
这是我参与11月更文挑战的第2天，活动详情查看：2021最后一次更文挑战

## 简介
在上篇文章中，了解了连接池的应用场景和本地运行了示例，本篇文章中，我们尝试来探索下Alibaba Druid数据库连接池的整体实现思路

## 连接池的大体实现思路
查询了阅读了一些资料：下面两篇还是写的比较好的

- [一文读懂连接池技术原理、设计与实现（Python）](https://toutiao.io/posts/nickqt/preview)
- [如何设计并实现一个db连接池？](https://juejin.cn/post/6844903853872119822)

文中实现一个数据库连接池的核心如下：

> 连接池设计的基本原理是这样的：
>（1）建立连接池对象（服务启动）。
>（2）按照事先指定的参数创建初始数量的连接（即：空闲连接数）。
>（3）对于一个访问请求，直接从连接池中得到一个连接。如果连接池对象中没有空闲的连接，且连接数没有达到最大（即：最大活跃连接数），创建一个新的连接；如果达到最大，则设定一定的超时时间，来获取连接。
>（4）运用连接访问服务。
>（5）访问服务完成，释放连接（此时的释放连接，并非真正关闭，而是将其放入空闲队列中。如实际空闲连接数大于初始空闲连接数则释放连接）。
>（6）释放连接池对象（服务停止、维护期间，释放连接池对象，并释放所有连接）。
> 摘抄自：[一文读懂连接池技术原理、设计与实现（Python）](https://toutiao.io/posts/nickqt/preview)

上面的配合上文中Python代码实现，感觉还是能体会到实现一个连接池的关键点的

## 源码探索
基于上面的思路，我们看出连接的获取和释放应该是实现的关键，下面我们开始尝试寻找下相关的代码

### 运行示例，debug入口
首先得找突破点，翻一翻test文件，找到下面一个东西：src/test/java/com/alibaba/druid/pool/demo/Demo0.java

```java
public class Demo0 extends TestCase {

    private String jdbcUrl;
    private String user;
    private String password;
    private String driverClass;
    private int    initialSize = 10;
    private int    minPoolSize = 1;
    private int    maxPoolSize = 2;
    private int    maxActive   = 12;

    protected void setUp() throws Exception {
        jdbcUrl = "jdbc:h2:file:./demo-db";
        user = "sa";
        password = "sa";
        driverClass = "org.h2.Driver";
    }

    public void test_0() throws Exception {
        DruidDataSource dataSource = new DruidDataSource();

        JMXUtils.register("com.alibaba:type=DruidDataSource", dataSource);

        dataSource.setInitialSize(initialSize);
        dataSource.setMaxActive(maxActive);
        dataSource.setMinIdle(minPoolSize);
        dataSource.setMaxIdle(maxPoolSize);
        dataSource.setPoolPreparedStatements(true);
        dataSource.setDriverClassName(driverClass);
        dataSource.setUrl(jdbcUrl);
        dataSource.setPoolPreparedStatements(true);
        dataSource.setUsername(user);
        dataSource.setPassword(password);
        dataSource.setValidationQuery("SELECT 1");
        dataSource.setTestOnBorrow(true);

        Connection conn = dataSource.getConnection();
        conn.close();

        System.out.println();
    }
}
```

源码中是不可以运行的，需要进行下面的修改才能成功运行起来：

- 1.将 maxActive 设置比 initSize 大：在运行中总是提示初始连接数最大活跃数，所以我们需要改下让初始连接数不小于最大活跃连接数
- 2.替换成内存数据（不替换也可以运行，这个只是单纯修改玩玩）

我们在下面的两行代码处都打上断点：

```java
        Connection conn = dataSource.getConnection();
        conn.close();
```

### 创建和获取数据库连接相关代码
根据断点进入：Connection conn = dataSource.getConnection(); 

```java
# DruidDataSource.java
    public DruidPooledConnection getConnection(long maxWaitMillis) throws SQLException {、
        # 初始化操作
        init();

        if (filters.size() > 0) {
            FilterChainImpl filterChain = new FilterChainImpl(this);
            return filterChain.dataSource_connect(this, maxWaitMillis);
        } else {
	    # 走到这里获取连接
            return getConnectionDirect(maxWaitMillis);
        }
    }
```

在上面的代码中直接去获取了初始化和数据库连接，我们先去瞅一瞅初始化相关的东西

在初始化的diam中，我们看到一些符合我们猜想的观点的类型队列初始化的东西，如下代码：

```java
# DruidDataSource.java
public void init() throws SQLException {
	    # 疑似初始化连接池
            connections = new DruidConnectionHolder[maxActive];
            evictConnections = new DruidConnectionHolder[maxActive];
            keepAliveConnections = new DruidConnectionHolder[maxActive];

            SQLException connectError = null;

            if (createScheduler != null && asyncInit) {
                for (int i = 0; i < initialSize; ++i) {
                    submitCreateTask(true);
                }
            } else if (!asyncInit) {
                // init connections
		// 这里就是根据配置初始化了初始数据量的连接
                while (poolingCount < initialSize) {
                    try {
                        PhysicalConnectionInfo pyConnectInfo = createPhysicalConnection();
                        DruidConnectionHolder holder = new DruidConnectionHolder(this, pyConnectInfo);
                        connections[poolingCount++] = holder;
                    } catch (SQLException ex) {
                        LOG.error("init datasource error, url: " + this.getUrl(), ex);
                        if (initExceptionThrow) {
                            connectError = ex;
                            break;
                        } else {
                            Thread.sleep(3000);
                        }
                    }
                }
            }
        } catch (SQLException e) {

        } finally {
            }
        }
    }
```

继续跟到获取连接

```java
public DruidPooledConnection getConnectionDirect(long maxWaitMillis) throws SQLException {
        int notFullTimeoutRetryCnt = 0;
        for (;;) {
            // handle notFullTimeoutRetry
            DruidPooledConnection poolableConnection;
            try {
		# 获取连接
                poolableConnection = getConnectionInternal(maxWaitMillis);
            } catch (GetConnectionTimeoutException ex) {
            }
            return poolableConnection;
        }
    }

private DruidPooledConnection getConnectionInternal(long maxWait) throws SQLException {
        DruidConnectionHolder holder;
	for (;;) {
                if (maxWait > 0) {
                    holder = pollLast(nanos);
                } else {
		    // 取最后一个
                    holder = takeLast();
                }

		// 相关统计的计算
                if (holder != null) {
                    if (holder.discard) {
                        continue;
                    }

                    activeCount++;
                    holder.active = true;
                    if (activeCount > activePeak) {
                        activePeak = activeCount;
                        activePeakTime = System.currentTimeMillis();
                    }
                }
            } catch (InterruptedException e) {
                connectErrorCountUpdater.incrementAndGet(this);
                throw new SQLException(e.getMessage(), e);
            } catch (SQLException e) {
                connectErrorCountUpdater.incrementAndGet(this);
                throw e;
            } finally {
                lock.unlock();
            }

            break;
        }

        DruidPooledConnection poolalbeConnection = new DruidPooledConnection(holder);
        return poolalbeConnection;
    }

DruidConnectionHolder takeLast() throws InterruptedException, SQLException {
        decrementPoolingCount();
        DruidConnectionHolder last = connections[poolingCount];
        connections[poolingCount] = null;

        return last;
    }
```

### 关闭连接

```java
# DruidPooledConnection.java
    @Override
    public void close() throws SQLException {
        try {
            List<Filter> filters = dataSource.getProxyFilters();
            if (filters.size() > 0) {
            } else {
                recycle();
            }
        } finally {
        }
    }

    public void recycle() throws SQLException {
        if (!this.abandoned) {
            DruidAbstractDataSource dataSource = holder.getDataSource();
            dataSource.recycle(this);
        }
    }
```

```java
# DruidDataSource.java
/**
     * 回收连接
     */
    protected void recycle(DruidPooledConnection pooledConnection) throws SQLException {
        final DruidConnectionHolder holder = pooledConnection.holder;
            lock.lock();
            try {
                if (holder.active) {
                    activeCount--;
                    holder.active = false;
                }
                closeCount++;

                result = putLast(holder, currentTimeMillis);
                recycleCount++;
            } finally {
                lock.unlock();
            }
    }
```