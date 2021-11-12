# Alibaba Druid 数据库连接池 连接关闭探索
***
这是我参与11月更文挑战的第5天，活动详情查看：2021最后一次更文挑战

## 简介
在上文中探索了数据库连接池的获取，下面接着初步来探索下数据库连接的关闭，看看其中具体执行了那些操作

## 连接关闭
下面的具体的代码，相关的位置，加上了自己的注释：

```java
    @Override
    public void close() throws SQLException {
        // 下面两段检查避免重复关闭
        if (this.disable) {
            return;
        }

        DruidConnectionHolder holder = this.holder;
        if (holder == null) {
            if (dupCloseLogEnable) {
                LOG.error("dup close");
            }
            return;
        }

        // 判断是否是一样的线程，不是则使用异步关闭
        DruidAbstractDataSource dataSource = holder.getDataSource();
        boolean isSameThread = this.getOwnerThread() == Thread.currentThread();

        if (!isSameThread) {
            dataSource.setAsyncCloseConnectionEnable(true);
        }

        if (dataSource.isAsyncCloseConnectionEnable()) {
            syncClose();
            return;
        }

        if (!CLOSING_UPDATER.compareAndSet(this, 0, 1)) {
            return;
        }

        try {
            // 连接关闭时的一些增强？
            for (ConnectionEventListener listener : holder.getConnectionEventListeners()) {
                listener.connectionClosed(new ConnectionEvent(this));
            }

            // Filter也能作用到连接关闭时?
            List<Filter> filters = dataSource.getProxyFilters();
            if (filters.size() > 0) {
                FilterChainImpl filterChain = new FilterChainImpl(dataSource);
                filterChain.dataSource_recycle(this);
            } else {
                recycle();
            }
        } finally {
            CLOSING_UPDATER.set(this, 0);
        }

        this.disable = true;
    }

    
    public void recycle() throws SQLException {
        if (this.disable) {
            return;
        }

        DruidConnectionHolder holder = this.holder;
        if (holder == null) {
            if (dupCloseLogEnable) {
                LOG.error("dup close");
            }
            return;
        }

        if (!this.abandoned) {
            DruidAbstractDataSource dataSource = holder.getDataSource();
            dataSource.recycle(this);
        }

        this.holder = null;
        conn = null;
        transactionInfo = null;
        closed = true;
    }
```

下面是具体的recycle()函数内容：

```java
    /**
     * 回收连接
     */
    protected void recycle(DruidPooledConnection pooledConnection) throws SQLException {
        final DruidConnectionHolder holder = pooledConnection.holder;

        if (holder == null) {
            LOG.warn("connectionHolder is null");
            return;
        }

        // 一个warn，并没有直接返回
        if (logDifferentThread //
            && (!isAsyncCloseConnectionEnable()) //
            && pooledConnection.ownerThread != Thread.currentThread()//
        ) {
            LOG.warn("get/close not same thread");
        }

        final Connection physicalConnection = holder.conn;

        // 使用双重检查锁，将traceEnable,目前尚不清楚其具体作用
        if (pooledConnection.traceEnable) {
            Object oldInfo = null;
            activeConnectionLock.lock();
            try {
                if (pooledConnection.traceEnable) {
                    oldInfo = activeConnections.remove(pooledConnection);
                    pooledConnection.traceEnable = false;
                }
            } finally {
                activeConnectionLock.unlock();
            }
            if (oldInfo == null) {
                if (LOG.isWarnEnabled()) {
                    LOG.warn("remove abandonded failed. activeConnections.size " + activeConnections.size());
                }
            }
        }

        final boolean isAutoCommit = holder.underlyingAutoCommit;
        final boolean isReadOnly = holder.underlyingReadOnly;
        final boolean testOnReturn = this.testOnReturn;

        try {
            // check need to rollback?
            if ((!isAutoCommit) && (!isReadOnly)) {
                pooledConnection.rollback();
            }

            // reset holder, restore default settings, clear warnings
            boolean isSameThread = pooledConnection.ownerThread == Thread.currentThread();
            if (!isSameThread) {
                final ReentrantLock lock = pooledConnection.lock;
                lock.lock();
                try {
                    holder.reset();
                } finally {
                    lock.unlock();
                }
            } else {
                holder.reset();
            }

            if (holder.discard) {
                return;
            }

            // 如果计数标识大于配置的最大数，则关闭连接
            if (phyMaxUseCount > 0 && holder.useCount >= phyMaxUseCount) {
                // 关闭了连接
                // 同时如果活跃连接数小于配置的最小活跃数，会发送通知（具体作用后面再查看）
                discardConnection(holder);
                return;
            }

            // 计数标识设置
            if (physicalConnection.isClosed()) {
                lock.lock();
                try {
                    if (holder.active) {
                        activeCount--;
                        holder.active = false;
                    }
                    closeCount++;
                } finally {
                    lock.unlock();
                }
                return;
            }

            // 检测连接是否还有效，无效则返还，并设置计数标识位
            if (testOnReturn) {
                boolean validate = testConnectionInternal(holder, physicalConnection);
                if (!validate) {
                    JdbcUtils.close(physicalConnection);

                    destroyCountUpdater.incrementAndGet(this);

                    lock.lock();
                    try {
                        if (holder.active) {
                            activeCount--;
                            holder.active = false;
                        }
                        closeCount++;
                    } finally {
                        lock.unlock();
                    }
                    return;
                }
            }
            if (holder.initSchema != null) {
                holder.conn.setSchema(holder.initSchema);
                holder.initSchema = null;
            }

            // 这里又关闭了一次连接
            // 在非enable的情况下
            if (!enable) {
                discardConnection(holder);
                return;
            }

            boolean result;
            final long currentTimeMillis = System.currentTimeMillis();

            // 连接超时关闭？这个超时具体是指什么还没搞清楚
            if (phyTimeoutMillis > 0) {
                long phyConnectTimeMillis = currentTimeMillis - holder.connectTimeMillis;
                if (phyConnectTimeMillis > phyTimeoutMillis) {
                    discardConnection(holder);
                    return;
                }
            }

            // 设置计数标识，同时没有关闭连接，而是放入池中
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

            if (!result) {
                JdbcUtils.close(holder.conn);
                LOG.info("connection recyle failed.");
            }
        } catch (Throwable e) {
            // 异常情况下关闭连接
            holder.clearStatementCache();

            if (!holder.discard) {
                discardConnection(holder);
                holder.discard = true;
            }

            LOG.error("recyle error", e);
            recycleErrorCountUpdater.incrementAndGet(this);
        }
    }
```

## 总结
通过上面代码可以看出：

- 1.Close做了同步和异步关闭之类的检查，但AsyncClose和recycle没有有比较大的区别，似乎也不是新