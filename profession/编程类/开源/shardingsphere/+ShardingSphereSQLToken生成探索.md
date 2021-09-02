# ShardingSphere SQLToken 生成探索
***
## 简介
在上篇文章中，我们探索了ShardingSphere中逻辑SQL变成真实SQL的解析过程，其中SQLToken在其中有着关键的作用，这篇文章中就探索下SQLToken的生成

## 源码分析
基于上文：

- [ShardingSphere 语句解析生成初探](https://juejin.cn/post/7003073129643769869)

我们继续看看SQLToken的生成

首先定位到SQLToken生成的关键代码部分：

```java
/**
 * SQL rewrite entry.
 */
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
	// 这里生成SQLToken
        result.generateSQLTokens();
        return result;
    }
    
    @SuppressWarnings({"unchecked", "rawtypes"})
    private void decorate(final Map<ShardingSphereRule, SQLRewriteContextDecorator> decorators, final SQLRewriteContext sqlRewriteContext, final RouteContext routeContext) {
        decorators.forEach((key, value) -> value.decorate(key, props, sqlRewriteContext, routeContext));
    }
}
```