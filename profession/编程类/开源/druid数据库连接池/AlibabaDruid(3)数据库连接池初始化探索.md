# Alibaba Druid 源码阅读（三） 数据库连接池初始化探索
***
这是我参与11月更文挑战的第1天，活动详情查看：2021最后一次更文挑战

## 简介
上文中探索了Alibaba Druid的连接池初始化和获取连接的关键代码，接下来详细看看初始化部分

## 数据库连接池初始化
对整个代码加上注释，如下：

```java
public void init() throws SQLException {
        // 已经初始化过了，直接返回
        if (inited) {
            return;
        }

        // bug fixed for dead lock, for issue #2980
        DruidDriver.getInstance();

        // 初始化时上锁
        final ReentrantLock lock = this.lock;
        try {
            lock.lockInterruptibly();
        } catch (InterruptedException e) {
            throw new SQLException("interrupt", e);
        }

        boolean init = false;
        try {
            if (inited) {
                return;
            }

            initStackTrace = Utils.toString(Thread.currentThread().getStackTrace());

            // 这部分尚不清楚作用
            this.id = DruidDriver.createDataSourceId();
            if (this.id > 1) {
                long delta = (this.id - 1) * 100000;
                this.connectionIdSeedUpdater.addAndGet(this, delta);
                this.statementIdSeedUpdater.addAndGet(this, delta);
                this.resultSetIdSeedUpdater.addAndGet(this, delta);
                this.transactionIdSeedUpdater.addAndGet(this, delta);
            }

            // 数据库连接url
            if (this.jdbcUrl != null) {
                this.jdbcUrl = this.jdbcUrl.trim();
                initFromWrapDriverUrl();
            }

            // filter相关，目前为空
            for (Filter filter : filters) {
                filter.init(this);
            }

            // 获取数据库类型
            if (this.dbTypeName == null || this.dbTypeName.length() == 0) {
                this.dbTypeName = JdbcUtils.getDbType(jdbcUrl, null);
            }

            // 针对 mysql、mariadb、oceanbase、ads 做其特有的处理
            DbType dbType = DbType.of(this.dbTypeName);
            if (dbType == DbType.mysql
                    || dbType == DbType.mariadb
                    || dbType == DbType.oceanbase
                    || dbType == DbType.ads) {
                boolean cacheServerConfigurationSet = false;
                if (this.connectProperties.containsKey("cacheServerConfiguration")) {
                    cacheServerConfigurationSet = true;
                } else if (this.jdbcUrl.indexOf("cacheServerConfiguration") != -1) {
                    cacheServerConfigurationSet = true;
                }
                if (cacheServerConfigurationSet) {
                    this.connectProperties.put("cacheServerConfiguration", "true");
                }
            }

            // 下面就是一些配置的效验
            // 这部分是不是放到最前面比较好？
            if (maxActive <= 0) {
                throw new IllegalArgumentException("illegal maxActive " + maxActive);
            }

            if (maxActive < minIdle) {
                throw new IllegalArgumentException("illegal maxActive " + maxActive);
            }

            if (getInitialSize() > maxActive) {
                throw new IllegalArgumentException("illegal initialSize " + this.initialSize + ", maxActive " + maxActive);
            }

            if (timeBetweenLogStatsMillis > 0 && useGlobalDataSourceStat) {
                throw new IllegalArgumentException("timeBetweenLogStatsMillis not support useGlobalDataSourceStat=true");
            }

            if (maxEvictableIdleTimeMillis < minEvictableIdleTimeMillis) {
                throw new SQLException("maxEvictableIdleTimeMillis must be grater than minEvictableIdleTimeMillis");
            }

            if (keepAlive && keepAliveBetweenTimeMillis <= timeBetweenEvictionRunsMillis) {
                throw new SQLException("keepAliveBetweenTimeMillis must be grater than timeBetweenEvictionRunsMillis");
            }

            // driverClass相关
            if (this.driverClass != null) {
                this.driverClass = driverClass.trim();
            }

            // 下面这段有些初始化加载和检查，但不是看的太明白
            initFromSPIServiceLoader();

            resolveDriver();

            initCheck();

            initExceptionSorter();
            initValidConnectionChecker();
            validationQueryCheck();

            // Stat 的作用是啥，目前尚不清楚
            if (isUseGlobalDataSourceStat()) {
                dataSourceStat = JdbcDataSourceStat.getGlobal();
                if (dataSourceStat == null) {
                    dataSourceStat = new JdbcDataSourceStat("Global", "Global", this.dbTypeName);
                    JdbcDataSourceStat.setGlobal(dataSourceStat);
                }
                if (dataSourceStat.getDbType() == null) {
                    dataSourceStat.setDbType(this.dbTypeName);
                }
            } else {
                dataSourceStat = new JdbcDataSourceStat(this.name, this.jdbcUrl, this.dbTypeName, this.connectProperties);
            }
            dataSourceStat.setResetStatEnable(this.resetStatEnable);

            // 关键的数据库连接池相关的，在获取连接中就是从DruidConnectionHolder中获取的
            connections = new DruidConnectionHolder[maxActive];
            evictConnections = new DruidConnectionHolder[maxActive];
            keepAliveConnections = new DruidConnectionHolder[maxActive];

            SQLException connectError = null;

            // 这段看大致意思的异步去生成数据库物理连接
            // 因为else中是可以看出直接获取了数据库的物理连接，然后放到DruidConnectionHolder中
            if (createScheduler != null && asyncInit) {
                for (int i = 0; i < initialSize; ++i) {
                    submitCreateTask(true);
                }
            } else if (!asyncInit) {
                // init connections
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

                if (poolingCount > 0) {
                    poolingPeak = poolingCount;
                    poolingPeakTime = System.currentTimeMillis();
                }
            }

            // 一些辅助性，日志啥的初始化操作
            createAndLogThread();
            createAndStartCreatorThread();
            createAndStartDestroyThread();

            initedLatch.await();
            init = true;

            initedTime = new Date();
            registerMbean();

            if (connectError != null && poolingCount == 0) {
                throw connectError;
            }

            // 前面使用的是initialSize 初始连接数
            // 这个使用的是 keepAlive 看着意思是空闲连接数
            if (keepAlive) {
                // async fill to minIdle
                if (createScheduler != null) {
                    for (int i = 0; i < minIdle; ++i) {
                        submitCreateTask(true);
                    }
                } else {
                    this.emptySignal();
                }
            }

        } catch (SQLException e) {
            LOG.error("{dataSource-" + this.getID() + "} init error", e);
            throw e;
        } catch (InterruptedException e) {
            throw new SQLException(e.getMessage(), e);
        } catch (RuntimeException e){
            LOG.error("{dataSource-" + this.getID() + "} init error", e);
            throw e;
        } catch (Error e){
            LOG.error("{dataSource-" + this.getID() + "} init error", e);
            throw e;

        } finally {
            inited = true;
            lock.unlock();

            if (init && LOG.isInfoEnabled()) {
                String msg = "{dataSource-" + this.getID();

                if (this.name != null && !this.name.isEmpty()) {
                    msg += ",";
                    msg += this.name;
                }

                msg += "} inited";

                LOG.info(msg);
            }
        }
    }
```

## 总结
从上看出：

- 初始化做了双检查锁
- 数据库可以异步或者同步产生配置的初始连接数
- 如果keepalive为true，异步生成配置的最小空闲连接数

当前只简单的了解了整个初始化过程，但其中还有很多细节，感觉是自己对这个业务场景没有理解到位