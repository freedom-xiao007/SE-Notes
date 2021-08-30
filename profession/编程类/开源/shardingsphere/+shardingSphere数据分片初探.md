# SharingSphere 源码解析 -- 数据分片初探
***
## 简介
在上一篇文章中，我们探索了ShardingSphere JDBC Mybatis示例执行的一个大致的过程，找到了SQL处理的关键节点，本文接下来就探索下数据库分片是如何生效的

## 源码解析
继续使用上篇文章中示例：[ShardingSphere JDBC 语句执行初探](https://juejin.cn/post/7001268789371207688)

在上文中： executionContext ，是一个非常关键的变量，可以说贯穿全文，经过我们调试发现，其已经具备真实的需要执行的语句，如下图：

![图片稍后上传]()

那真实SQL生成的关键代码就如下,也就是上文中与Mybatis承接的部分

```java
# ShardingSpherePreparedStatement.java
    @Override
    public boolean execute() throws SQLException {
        try {
            clearPrevious();
	    // 在这就已经生成了真实的SQL等语句
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
            ExecutionGroupContext<JDBCExecutionUnit> executionGroupContext = createExecutionGroupContext();
            cacheStatements(executionGroupContext.getInputGroups());
            return driverJDBCExecutor.execute(executionGroupContext,
                    executionContext.getLogicSQL(), executionContext.getRouteContext().getRouteUnits(), createExecuteCallback());
        } finally {
            clearBatch();
        }
    }
```

如上，下面我们就开始跟踪生成语句： executionContext = createExecutionContext();

```java
    private ExecutionContext createExecutionContext() {
        LogicSQL logicSQL = createLogicSQL();
        SQLCheckEngine.check(logicSQL.getSqlStatementContext().getSqlStatement(), logicSQL.getParameters(), 
                metaDataContexts.getMetaData(connection.getSchemaName()).getRuleMetaData().getRules(), connection.getSchemaName(), metaDataContexts.getMetaDataMap(), null);
	// 这生成真实的SQL
        ExecutionContext result = kernelProcessor.generateExecutionContext(logicSQL, metaDataContexts.getMetaData(connection.getSchemaName()), metaDataContexts.getProps());
        findGeneratedKey(result).ifPresent(generatedKey -> generatedValues.addAll(generatedKey.getGeneratedValues()));
        return result;
    }
```

跟踪来到下面的关键类，通过注释，我们看到其持有原始的SQL，相关的配置信息，生成了我们想要的 execution context

```java
public final class KernelProcessor {
    
    /**
     * Generate execution context.
     *
     * @param logicSQL logic SQL
     * @param metaData ShardingSphere meta data
     * @param props configuration properties
     * @return execution context
     */
    public ExecutionContext generateExecutionContext(final LogicSQL logicSQL, final ShardingSphereMetaData metaData, final ConfigurationProperties props) {
        RouteContext routeContext = route(logicSQL, metaData, props);
        SQLRewriteResult rewriteResult = rewrite(logicSQL, metaData, props, routeContext);
        ExecutionContext result = createExecutionContext(logicSQL, metaData, routeContext, rewriteResult);
        logSQL(logicSQL, props, result);
        return result;
    }
}
```

继续跟下去：