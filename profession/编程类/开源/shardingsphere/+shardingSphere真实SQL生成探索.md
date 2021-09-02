# SharingSphere 源码解析 -- 真实SQL生成探索
***
## 简介
在上一篇文章中，我们探索了ShardingSphere JDBC Mybatis示例执行的一个大致的过程，找到了SQL处理的关键节点，看看一个逻辑的SQL变成真实SQL有哪些关键点

## 源码解析
注：一番探索下来，逻辑SQL达到真实数据库的处理有点复杂了，需要花不少时间debug，放在一篇也太长了，所以进行拆分，这样简单点

### SQL解析生成入口
继续使用上篇文章中示例：[ShardingSphere JDBC 语句执行初探](https://juejin.cn/post/7001268789371207688)

在上文中： executionContext ，是一个非常关键的变量，可以说贯穿全文，经过我们调试发现，其已经具备真实的需要执行的语句，如下图：


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5fecf638281f4b5991afd52774128a94~tplv-k3u1fbpfcp-watermark.image)

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

### SQL的解析生成路径
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

上面这个函数非常非常的核心，下面看看其三个关键的变量： routeContext, rewriteResult, result

routeContext 从名字可以看出，其应该是保存了逻辑SQL到真实SQL相关路由信息，通过debug我们也可以看出，如下图：


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/55c598066acb4ebab2d512e6e8e56d16~tplv-k3u1fbpfcp-watermark.image)

从上面标红的东西可以看到，其已经有了数据库和表的相关的路由的信息，很关键

rewriteResult 经过重写的SQL，得到了真实的SQL语句，如下图：


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1460be482f16465ba43f2f8fa8afc21d~tplv-k3u1fbpfcp-watermark.image)

result 拼装 routeContext, rewriteResult 得到了一直贯穿整个ShardingSphere语句执行的东西，这个前面的文章和本文的开篇都有涉及，这里就不再赘述了

在： SQLRewriteResult rewriteResult = rewrite(logicSQL, metaData, props, routeContext) 继续跟下去：

```java
public final class SQLRewriteEntry {
    
    /**
     * Rewrite.
     * 
     * @param sql SQL
     * @param parameters SQL parameters
     * @param sqlStatementContext SQL statement context
     * @param routeContext route context
     * @return route unit and SQL rewrite result map
     */
    public SQLRewriteResult rewrite(final String sql, final List<Object> parameters, final SQLStatementContext<?> sqlStatementContext, final RouteContext routeContext) {
        SQLRewriteContext sqlRewriteContext = createSQLRewriteContext(sql, parameters, sqlStatementContext, routeContext);
        return routeContext.getRouteUnits().isEmpty()
                ? new GenericSQLRewriteEngine().rewrite(sqlRewriteContext) : new RouteSQLRewriteEngine().rewrite(sqlRewriteContext, routeContext);
    }
    
    private SQLRewriteContext createSQLRewriteContext(final String sql, final List<Object> parameters, final SQLStatementContext<?> sqlStatementContext, final RouteContext routeContext) {
        SQLRewriteContext result = new SQLRewriteContext(schema, sqlStatementContext, sql, parameters);
        decorate(decorators, result, routeContext);
        result.generateSQLTokens();
        return result;
    }
}
```

在上面的: createSQLRewriteContext 函数中，其实还没有生成真实的SQL，但其生成非常关键的用于后面真实SQL的关键元信息，这部分这里不细说，后面再研究

我们本次走入处理分支： new RouteSQLRewriteEngine().rewrite(sqlRewriteContext, routeContext)

目前我们是有 route，但如果我们没有route，走： new GenericSQLRewriteEngine().rewrite(sqlRewriteContext) 是什么样的场景呢？这里标记一个，我们先继续跟着主线看：

```java
public final class RouteSQLRewriteEngine {
    
    /**
     * Rewrite SQL and parameters.
     *
     * @param sqlRewriteContext SQL rewrite context
     * @param routeContext route context
     * @return SQL rewrite result
     */
    public RouteSQLRewriteResult rewrite(final SQLRewriteContext sqlRewriteContext, final RouteContext routeContext) {
        Map<RouteUnit, SQLRewriteUnit> result = new LinkedHashMap<>(routeContext.getRouteUnits().size(), 1);
        for (RouteUnit each : routeContext.getRouteUnits()) {
            result.put(each, new SQLRewriteUnit(new RouteSQLBuilder(sqlRewriteContext, each).toSQL(), getParameters(sqlRewriteContext.getParameterBuilder(), routeContext, each)));
        }
        return new RouteSQLRewriteResult(result);
    }
}
```

上面的代码中，类：SQLRewriteUnit，直接由一个关键的变量：sql，这个是我们想要的真实的SQL，而其就是由本次最终的核心代码生成的：new SQLRewriteUnit(new RouteSQLBuilder(sqlRewriteContext, each).toSQL(),其toSQL函数就是真实SQL生成的位置：

```java
public abstract class AbstractSQLBuilder implements SQLBuilder {
    
    private final SQLRewriteContext context;
    
    @Override
    public final String toSQL() {
        if (context.getSqlTokens().isEmpty()) {
            return context.getSql();
        }
        Collections.sort(context.getSqlTokens());
        StringBuilder result = new StringBuilder();
        result.append(context.getSql(), 0, context.getSqlTokens().get(0).getStartIndex());
        for (SQLToken each : context.getSqlTokens()) {
            result.append(each instanceof ComposableSQLToken ? getComposableSQLTokenText((ComposableSQLToken) each) : getSQLTokenText(each));
            result.append(getConjunctionText(each));
        }
        return result.toString();
    }
    
    protected abstract String getSQLTokenText(SQLToken sqlToken);

    private String getComposableSQLTokenText(final ComposableSQLToken composableSQLToken) {
        StringBuilder result = new StringBuilder();
        for (SQLToken each : composableSQLToken.getSqlTokens()) {
            result.append(getSQLTokenText(each));
            result.append(getConjunctionText(each));
        }
        return result.toString();
    }

    private String getConjunctionText(final SQLToken sqlToken) {
        return context.getSql().substring(getStartIndex(sqlToken), getStopIndex(sqlToken));
    }
}
```

通过debug，我们慢慢的看到变量：result，逐渐的变成我们想要的真实的SQL，到这里我们基本找到了路径了

这里逻辑还是有点多，还没分析完，就留到下篇分解了

## 总结
总结下我们这边探索的真实SQL经历的关键节点

- ShardingSpherePreparedStatement.java： executionContext = createExecutionContext();
- ShardingSpherePreparedStatement.java： ExecutionContext result = kernelProcessor.generateExecutionContext(logicSQL, metaDataContexts.getMetaData(connection.getSchemaName()), metaDataContexts.getProps());
- KernelProcessor.java: generateExecutionContext
  - RouteContext routeContext = route(logicSQL, metaData, props);
  - SQLRewriteResult rewriteResult = rewrite(logicSQL, metaData, props, routeContext);
    - SQLRewriteEntry：rewrite
      - new GenericSQLRewriteEngine().rewrite(sqlRewriteContext)
      - new RouteSQLRewriteEngine().rewrite(sqlRewriteContext, routeContext)
  - ExecutionContext result = createExecutionContext(logicSQL, metaData, routeContext, rewriteResult)

本次其实就定位了具体的真实SQL的关键代码，逻辑梳理还是比较花时间的，所以就留到下一篇讲解了