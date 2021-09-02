# ShardingSphere JDBC 语句执行初探
***
## 简介
在前几篇文中，我们基于源码就ShardingSphere的核心功能给运行了一遍，本篇文章开始，我们开始探索源码，看看ShardingSphere是如何进行工作的

## 概览
开始之前，我们先思考这次探索的疑点：

- 1.ShardingSphere是如何加载我们配置的多个数据库源的？
- 2.ShardingSphere是如何将语句写到不同的数据源的？

带着问题去探索，虽然可能得不到答案，但起码有个方向

基于我们JDBC探索的文章，打上相关的断点，开始我们的探索之旅

- [ShardingSphere JDBC 分库分表 读写分离 数据加密 ](https://juejin.cn/post/6999625443930439693/)

## 源码探索
### 1.寻找ShardingSphere JDBC 的入口
在下面代码中的init和process打上断点，看看能不能通过程序调用栈找到蛛丝马迹

```java
public final class ExampleExecuteTemplate {
    
    public static void run(final ExampleService exampleService) throws SQLException {
        try {
            exampleService.initEnvironment();
            exampleService.processSuccess();
        } finally {
            exampleService.cleanEnvironment();
        }
    }
}
```

exampleService.initEnvironment() 进来以后，调用栈空空如也，啥也没有，看来我们第一个探索如果加载多个配置的数据源的目的告吹了

但还是继续Debug下去，竟然全是Ibatis相关的东西，没有Debug到ShardingSphere相关的代码，安慰自己，正常现象

继续探索：exampleService.processSuccess()，来到下面的相关语句的插入、查询等相关的操作，在insertData()打上断点

```java
public class OrderServiceImpl implements ExampleService {
    @Override
    @Transactional
    public void processSuccess() throws SQLException {
        System.out.println("-------------- Process Success Begin ---------------");
        List<Long> orderIds = insertData();
        printData();
        deleteData(orderIds);
        printData();
        System.out.println("-------------- Process Success Finish --------------");
    }
}
```

基于之前尝试没有收获，这次不得不小心翼翼，跟着函数调用不断的深入下去

MapperProxy.class，执行下面的方法，进行跟下去：

```java
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            if (Object.class.equals(method.getDeclaringClass())) {
                return method.invoke(this, args);
            }

            if (this.isDefaultMethod(method)) {
                return this.invokeDefaultMethod(proxy, method, args);
            }
        } catch (Throwable var5) {
            throw ExceptionUtil.unwrapThrowable(var5);
        }

        MapperMethod mapperMethod = this.cachedMapperMethod(method);
	// 从这里进入
        return mapperMethod.execute(this.sqlSession, args);
    }
```

来到下面的函数

```java
    public Object execute(SqlSession sqlSession, Object[] args) {
        Object result;
        Object param;
        switch(this.command.getType()) {
        case INSERT:
            param = this.method.convertArgsToSqlCommandParam(args);
	    // 从这里进入，跟踪insert方法
            result = this.rowCountResult(sqlSession.insert(this.command.getName(), param));
            break;
        case UPDATE:
	......
    }
```

其中我们看到了熟悉的数据库相关的参数：SqlSession sqlSession，我们调用堆栈，查看其内容，如下图：


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/939246803c2642c2a679b6bfe16ff81f~tplv-k3u1fbpfcp-watermark.image)

数据库名称竟然是 logic_db，想到秦老师讲课时说ShardingSphere UI中也能用JDBC，只是它的名称是 logic_db

此时有了一些猜测，但得继续跟踪下去，证实下

来到了：SqlSessionTemplate.class，其中重新获取了一次 sqlSession，但其实还是logic_db，继续跟

```java
private class SqlSessionInterceptor implements InvocationHandler {
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            SqlSession sqlSession = SqlSessionUtils.getSqlSession(SqlSessionTemplate.this.sqlSessionFactory, SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);

            Object unwrapped;
            try {
		// 这里继续跟下去
                Object result = method.invoke(sqlSession, args);
                if (!SqlSessionUtils.isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
                    sqlSession.commit(true);
                }

                unwrapped = result;
            } catch (Throwable var11) {
               ......
            } finally {
               ......
            }
            return unwrapped;
        }
    }
```

一路上遇到很多的 update 操作，我们进入跟下去即可，最后我们来到：PreparedStatementHandler.class

```java
public class PreparedStatementHandler extends BaseStatementHandler {
    public int update(Statement statement) throws SQLException {
        PreparedStatement ps = (PreparedStatement)statement;
	// 执行这里很关键了
        ps.execute();
        int rows = ps.getUpdateCount();
        Object parameterObject = this.boundSql.getParameterObject();
        KeyGenerator keyGenerator = this.mappedStatement.getKeyGenerator();
        keyGenerator.processAfter(this.executor, this.mappedStatement, ps, parameterObject);
        return rows;
    }
}
```

在上面的：ps.execute(),我们跟下去的时候，惊喜就来了，砰的一下，就来到了ShardingSphere自己写的类中：ShardingSpherePreparedStatement.java

到这里，对于ShardingSphere JDBC有了一些想法：

ShardingSphere JDBC 感觉其实本质上还有一个中间件，内嵌式的，同样是介于程序与数据库之间，在本地探索中，它就以一个 logic_db ，最为 Mybatis的下游，将所有的数据库进行获取截断处理

初步有这么个体会，后面随着研究的深入可能会有更多的发现，到这我们就找到了ShardingSphere的入口了

### 2.初步探索ShardingSphere JDBC 处理路径
在上面我们来到了这次找到的入口：ShardingSpherePreparedStatement.java

```java
    @Override
    public boolean execute() throws SQLException {
        try {
            clearPrevious();
	    // 这里有了真实的数据库源和语句
            executionContext = createExecutionContext();
            if (metaDataContexts.getMetaData(connection.getSchemaName()).getRuleMetaData().getRules().stream().anyMatch(each -> each instanceof RawExecutionRule)) {
                // TODO process getStatement
                Collection<ExecuteResult> executeResults = rawExecutor.execute(createRawExecutionGroupContext(), executionContext.getLogicSQL(), new RawSQLExecutorCallback());
                return executeResults.iterator().next() instanceof QueryResult;
            }
            if (executionContext.getRouteContext().isFederated()) {
                List<QueryResult> queryResults = executeFederatedQuery();
                return !queryResults.isEmpty();
            }
	    // 继续跟踪
            ExecutionGroupContext<JDBCExecutionUnit> executionGroupContext = createExecutionGroupContext();
            cacheStatements(executionGroupContext.getInputGroups());
            return driverJDBCExecutor.execute(executionGroupContext,
                    executionContext.getLogicSQL(), executionContext.getRouteContext().getRouteUnits(), createExecuteCallback());
        } finally {
            clearBatch();
        }
    }
```

executionContext = createExecutionContext() 有原始的语句，还是我们定时真实的数据源和SQL语句，如下图，我们继续跟下去：


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5e774d6347044d8bba23e9d59002b3e1~tplv-k3u1fbpfcp-watermark.image)

来到：AbstractExecutionPrepareEngine.java,前面其实已经能得到展示的执行语句，目前还不太清楚这步的目的是啥

```java
    public final ExecutionGroupContext<T> prepare(final RouteContext routeContext, final Collection<ExecutionUnit> executionUnits) throws SQLException {
        Collection<ExecutionGroup<T>> result = new LinkedList<>();
        for (Entry<String, List<SQLUnit>> entry : aggregateSQLUnitGroups(executionUnits).entrySet()) {
            String dataSourceName = entry.getKey();
            List<SQLUnit> sqlUnits = entry.getValue();
            List<List<SQLUnit>> sqlUnitGroups = group(sqlUnits);
            ConnectionMode connectionMode = maxConnectionsSizePerQuery < sqlUnits.size() ? ConnectionMode.CONNECTION_STRICTLY : ConnectionMode.MEMORY_STRICTLY;
            result.addAll(group(dataSourceName, sqlUnitGroups, connectionMode));
        }
        return decorate(routeContext, result);
    }
```

回到 ShardingSpherePreparedStatement.java，跟踪 driverJDBCExecutor.execute，来到：DriverJDBCExecutor.java

```java
   public boolean execute(final ExecutionGroupContext<JDBCExecutionUnit> executionGroupContext, final LogicSQL logicSQL,
                           final Collection<RouteUnit> routeUnits, final JDBCExecutorCallback<Boolean> callback) throws SQLException {
        try {
            ExecuteProcessEngine.initialize(logicSQL, executionGroupContext, metaDataContexts.getProps());
            List<Boolean> results = jdbcLockEngine.execute(executionGroupContext, logicSQL.getSqlStatementContext(), routeUnits, callback);
            boolean result = null != results && !results.isEmpty() && null != results.get(0) && results.get(0);
            ExecuteProcessEngine.finish(executionGroupContext.getExecutionID());
            return result;
        } finally {
            ExecuteProcessEngine.clean();
        }
    }
```

有趣的是 jdbcLockEngine 还是 logic_db，继续跟踪execute函数，一路跟下去来到： ExecutorEngine.java

```java
    public <I, O> List<O> execute(final ExecutionGroupContext<I> executionGroupContext,
                                  final ExecutorCallback<I, O> firstCallback, final ExecutorCallback<I, O> callback, final boolean serial) throws SQLException {
        if (executionGroupContext.getInputGroups().isEmpty()) {
            return Collections.emptyList();
        }
        return serial ? serialExecute(executionGroupContext.getInputGroups().iterator(), firstCallback, callback)
                : parallelExecute(executionGroupContext.getInputGroups().iterator(), firstCallback, callback);
    }

    private <I, O> List<O> serialExecute(final Iterator<ExecutionGroup<I>> executionGroups, final ExecutorCallback<I, O> firstCallback, final ExecutorCallback<I, O> callback) throws SQLException {
        ExecutionGroup<I> firstInputs = executionGroups.next();
        List<O> result = new LinkedList<>(syncExecute(firstInputs, null == firstCallback ? callback : firstCallback));
        while (executionGroups.hasNext()) {
	    // 继续跟踪
            result.addAll(syncExecute(executionGroups.next(), callback));
        }
        return result;
    }
```

一路跟踪下去，来到： JDBCExecutorCallback.java

```java
    public final Collection<T> execute(final Collection<JDBCExecutionUnit> executionUnits, final boolean isTrunkThread, final Map<String, Object> dataMap) throws SQLException {
        // TODO It is better to judge whether need sane result before execute, can avoid exception thrown
        Collection<T> result = new LinkedList<>();
        for (JDBCExecutionUnit each : executionUnits) {
            T executeResult = execute(each, isTrunkThread, dataMap);
            if (null != executeResult) {
                result.add(executeResult);
            }
        }
        return result;
    }

    private T execute(final JDBCExecutionUnit jdbcExecutionUnit, final boolean isTrunkThread, final Map<String, Object> dataMap) throws SQLException {
        SQLExecutorExceptionHandler.setExceptionThrown(isExceptionThrown);
	// 在这里得到了我们真实的数据库源
        DataSourceMetaData dataSourceMetaData = getDataSourceMetaData(jdbcExecutionUnit.getStorageResource().getConnection().getMetaData());
        SQLExecutionHook sqlExecutionHook = new SPISQLExecutionHook();
        try {
            SQLUnit sqlUnit = jdbcExecutionUnit.getExecutionUnit().getSqlUnit();
            sqlExecutionHook.start(jdbcExecutionUnit.getExecutionUnit().getDataSourceName(), sqlUnit.getSql(), sqlUnit.getParameters(), dataSourceMetaData, isTrunkThread, dataMap);
	    // 继续跟踪执行语句
            T result = executeSQL(sqlUnit.getSql(), jdbcExecutionUnit.getStorageResource(), jdbcExecutionUnit.getConnectionMode());
            sqlExecutionHook.finishSuccess();
            finishReport(dataMap, jdbcExecutionUnit);
            return result;
        } catch (final SQLException ex) {
	    ......
        }
    }
```

在上面的代码中，找到直接获取真实数据库源相关的代码


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/692bd86a53e24508b34b9e3d73f1c7d4~tplv-k3u1fbpfcp-watermark.image)

我们继续跟踪执行的相关代码： ShardingSpherePreparedStatement.java ， 来到

```java
   private JDBCExecutorCallback<Boolean> createExecuteCallback() {
        boolean isExceptionThrown = SQLExecutorExceptionHandler.isExceptionThrown();
        return new JDBCExecutorCallback<Boolean>(metaDataContexts.getMetaData(connection.getSchemaName()).getResource().getDatabaseType(), sqlStatement, isExceptionThrown) {
            
            @Override
            protected Boolean executeSQL(final String sql, final Statement statement, final ConnectionMode connectionMode) throws SQLException {
                return ((PreparedStatement) statement).execute();
            }
            
            @Override
            protected Optional<Boolean> getSaneResult(final SQLStatement sqlStatement) {
                return Optional.empty();
            }
        };
    }
```

其中的: Statement statement ,如果用原生的 JDBC 自己写过操作数据库，那应该比较数据，那语句执行到这里就基本告一个段落了

## 总结
这次感觉收获还是挺大，虽然相关的数据库源的解析和语句如何打到对应的数据库的部分没有探索，但我们已经定位到了其大致的位置，可以后面继续研究

ShardingSphere JDBC 在本次探索中，感觉就是一个内嵌的 Proxy ，其数据库库名固定为 logic_db，截断了 Mybatis与真实数据库的链接，当前了中间商，大致如下图：

![ShardingSphereJDBC语句执行初探.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9cffed1bfde43099d0555f0b6857851~tplv-k3u1fbpfcp-watermark.image)