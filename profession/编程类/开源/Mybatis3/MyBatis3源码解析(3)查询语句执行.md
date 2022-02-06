# MyBatis3源码解析(3)查询语句执行
***
这是我参与2022首次更文挑战的第8天，活动详情查看：2022首次更文挑战

## 简介
上篇探索了MyBatis中如何获取数据库连接，本篇继续探索，来看看MyBatis中如何执行一条查询语句

## 测试代码
本篇文中用于调试的测试代码请参考：[MyBatis3源码解析（1）探索准备](https://juejin.cn/post/7058354949209456653)

完整的工程已放到GitHub上：https://github.com/lw1243925457/MybatisDemo/tree/master/example

## 源码解析
在我们的日常开发中，Mapper类我们往往就是定义了一个接口，在其中写方法和对应的语句，然后我们就能直接使用了，很方便，而这背后的原理就是JDK代理

如下所示，当我们执行语句：Person person = personMapper.getPersonById(1);

### MapperProxy
就来到了Mapper的代理类：MapperProxy，具体的执行逻辑就在invoke函数中

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {
    public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperProxy.MapperMethodInvoker> methodCache) {
        this.sqlSession = sqlSession;
        this.mapperInterface = mapperInterface;
        this.methodCache = methodCache;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            return Object.class.equals(method.getDeclaringClass()) ? method.invoke(this, args) : this.cachedInvoker(method).invoke(proxy, method, args, this.sqlSession);
        } catch (Throwable var5) {
            throw ExceptionUtil.unwrapThrowable(var5);
        }
    }

    private MapperProxy.MapperMethodInvoker cachedInvoker(Method method) throws Throwable {
        try {
            return (MapperProxy.MapperMethodInvoker)MapUtil.computeIfAbsent(this.methodCache, method, (m) -> {
                if (m.isDefault()) {
                    try {
                        return privateLookupInMethod == null ? new MapperProxy.DefaultMethodInvoker(this.getMethodHandleJava8(method)) : new MapperProxy.DefaultMethodInvoker(this.getMethodHandleJava9(method));
                    } catch (InstantiationException | InvocationTargetException | NoSuchMethodException | IllegalAccessException var4) {
                        throw new RuntimeException(var4);
                    }
                } else {
		    // 本地调试第一次执行，走这里的逻辑
                    return new MapperProxy.PlainMethodInvoker(new MapperMethod(this.mapperInterface, method, this.sqlSession.getConfiguration()));
                }
            });
        } catch (RuntimeException var4) {
            Throwable cause = var4.getCause();
            throw (Throwable)(cause == null ? var4 : cause);
        }
    }
}
```

值得注意的一点是，SQLSession包含数据库的连接信息，也就是数据库的相关东西是提前初始化的

### MapperMethod
接着来到MapperMethod中，其中我们可以看到：

根据不同的SQL类型，走不同的类型

在其中的SELECT中，根据不同的数据返回类型，走不同的逻辑

```java
public class MapperMethod {
    public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
        this.command = new MapperMethod.SqlCommand(config, mapperInterface, method);
        this.method = new MapperMethod.MethodSignature(config, mapperInterface, method);
    }

    public Object execute(SqlSession sqlSession, Object[] args) {
        Object result;
        Object param;
        switch(this.command.getType()) {
        case INSERT:
            param = this.method.convertArgsToSqlCommandParam(args);
            result = this.rowCountResult(sqlSession.insert(this.command.getName(), param));
            break;
        case UPDATE:
            param = this.method.convertArgsToSqlCommandParam(args);
            result = this.rowCountResult(sqlSession.update(this.command.getName(), param));
            break;
        case DELETE:
            param = this.method.convertArgsToSqlCommandParam(args);
            result = this.rowCountResult(sqlSession.delete(this.command.getName(), param));
            break;
        case SELECT:
	    // 根据不同的返回类型，走不同的逻辑
	    // 但其中调用的解析参数的方法是一样的
            if (this.method.returnsVoid() && this.method.hasResultHandler()) {
                this.executeWithResultHandler(sqlSession, args);
                result = null;
            } else if (this.method.returnsMany()) {
                result = this.executeForMany(sqlSession, args);
            } else if (this.method.returnsMap()) {
                result = this.executeForMap(sqlSession, args);
            } else if (this.method.returnsCursor()) {
                result = this.executeForCursor(sqlSession, args);
            } else {
		// 解析参数的方法，上面的查询处理逻辑中也调用了此方法
		// 也就是这个就是通用的参数处理逻辑
                param = this.method.convertArgsToSqlCommandParam(args);
                result = sqlSession.selectOne(this.command.getName(), param);
                if (this.method.returnsOptional() && (result == null || !this.method.getReturnType().equals(result.getClass()))) {
                    result = Optional.ofNullable(result);
                }
            }
            break;
        case FLUSH:
            result = sqlSession.flushStatements();
            break;
        default:
            throw new BindingException("Unknown execution method for: " + this.command.getName());
        }

        if (result == null && this.method.getReturnType().isPrimitive() && !this.method.returnsVoid()) {
            throw new BindingException("Mapper method '" + this.command.getName() + " attempted to return null from a method with a primitive return type (" + this.method.getReturnType() + ").");
        } else {
            return result;
        }
    }
}
```

其中包含了两个重要的变量，我们来看看其中的内容：

method:

- returnsMany=false
- returnsMap=false
- returnsVoid=false
- returnsCursor=false
- returnsOptional=false
- returnsType=Person
- mapKey=null
- resultHandlerIndex=null
- rowBoundsIndex=null
- paramNameResolver

可以看到其中包含的查询参数相关的，还有很多与返回结果相关的，返回结果相关的大部分都是可设置的，这个后面我们尝试探索探索

command：

- name=mapper.PersonMapper.getPersonById
- type=SELECT

command中就name是调用方法的具体名称，可以说是唯一标识，而Type是语句类型

command是提前就生成好的，这部分涉及到Mapper的初始化内容，后面再进行探索

上面的关键逻辑有参数的解析，这部分在日常的开发接触也较多，后面将用一篇文章来详细探索下

这里获得的Param就是：1

### SQLSession
接下来就到了MyBatis语句执行相关的

```java
public class DefaultSqlSession implements SqlSession {
    public <T> T selectOne(String statement, Object parameter) {
        List<T> list = this.selectList(statement, parameter);
        if (list.size() == 1) {
            return list.get(0);
        } else if (list.size() > 1) {
            throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
        } else {
            return null;
        }
    }

    public <E> List<E> selectList(String statement, Object parameter) {
        return this.selectList(statement, parameter, RowBounds.DEFAULT);
    }

    public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
        return this.selectList(statement, parameter, rowBounds, Executor.NO_RESULT_HANDLER);
    }

    
    private <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
        List var6;
        try {
	    // 已初始化好的
            MappedStatement ms = this.configuration.getMappedStatement(statement);
	    // 执行SQL语句
            var6 = this.executor.query(ms, this.wrapCollection(parameter), rowBounds, handler);
        } catch (Exception var10) {
            throw ExceptionFactory.wrapException("Error querying database.  Cause: " + var10, var10);
        } finally {
            ErrorContext.instance().reset();
        }

        return var6;
    }
}
```

### Executor
然后来到Executor，之类首先来到Cache，然后到Simple，Cache的作用后面我们再进行分析

```java
public class CachingExecutor implements Executor {
    public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
        BoundSql boundSql = ms.getBoundSql(parameterObject);
        CacheKey key = this.createCacheKey(ms, parameterObject, rowBounds, boundSql);
        return this.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
    }

    public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
        Cache cache = ms.getCache();
        if (cache != null) {
            this.flushCacheIfRequired(ms);
            if (ms.isUseCache() && resultHandler == null) {
                this.ensureNoOutParams(ms, boundSql);
                List<E> list = (List)this.tcm.getObject(cache, key);
                if (list == null) {
                    list = this.delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                    this.tcm.putObject(cache, key, list);
                }

                return list;
            }
        }

        return this.delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
    }

    public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
        ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
        if (this.closed) {
            throw new ExecutorException("Executor was closed.");
        } else {
            if (this.queryStack == 0 && ms.isFlushCacheRequired()) {
                this.clearLocalCache();
            }

            List list;
            try {
                ++this.queryStack;
		// 从local中获取缓存结果，但作用域是多少呢？这个后面我们探索下
                list = resultHandler == null ? (List)this.localCache.getObject(key) : null;
                if (list != null) {
                    this.handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
                } else {
		    // 查询
                    list = this.queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
                }
            } finally {
                --this.queryStack;
            }
	    .......

            return list;
        }
    }

    private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
        this.localCache.putObject(key, ExecutionPlaceholder.EXECUTION_PLACEHOLDER);

        List list;
        try {
            list = this.doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
        } finally {
            this.localCache.removeObject(key);
        }

        // 取得结果后，放入缓存
        this.localCache.putObject(key, list);
        if (ms.getStatementType() == StatementType.CALLABLE) {
            this.localOutputParameterCache.putObject(key, parameter);
        }

        return list;
    }
}

public class SimpleExecutor extends BaseExecutor {
    public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
        Statement stmt = null;

        List var9;
        try {
            Configuration configuration = ms.getConfiguration();
            StatementHandler handler = configuration.newStatementHandler(this.wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
	    // 到这里就得到已经能够执行的stmt：
	    // prep2: select id,name from person where id=?{1: 1}
            stmt = this.prepareStatement(handler, ms.getStatementLog());
            var9 = handler.query(stmt, resultHandler);
        } finally {
            this.closeStatement(stmt);
        }

        return var9;
    }

    private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
        Connection connection = this.getConnection(statementLog);
        Statement stmt = handler.prepare(connection, this.transaction.getTimeout());
        handler.parameterize(stmt);
        return stmt;
    }
}
```

### StatementHandler
这里是MyBatis的StatementHandler，直接拿到我们的stmt执行，得到结果

```java
public class PreparedStatementHandler extends BaseStatementHandler {
    public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
        PreparedStatement ps = (PreparedStatement)statement;
        ps.execute();
        return this.resultSetHandler.handleResultSets(ps);
    }
}
```

### ResultHandler
得到结果后，获取结果，并处理转换成我们想要的结果，比如我们这里的是Person类

获取结果并处理转换，这个也是我们日常开发中经常打交道的，自己前期也遇到不少的问题

这部分后面也会用单独的篇章进行详细的分析

```java
public class DefaultResultSetHandler implements ResultSetHandler {
    public List<Object> handleResultSets(Statement stmt) throws SQLException {
        ErrorContext.instance().activity("handling results").object(this.mappedStatement.getId());
        List<Object> multipleResults = new ArrayList();
        int resultSetCount = 0;
        ResultSetWrapper rsw = this.getFirstResultSet(stmt);
        List<ResultMap> resultMaps = this.mappedStatement.getResultMaps();
        int resultMapCount = resultMaps.size();
        this.validateResultMapsCount(rsw, resultMapCount);

        while(rsw != null && resultMapCount > resultSetCount) {
            ResultMap resultMap = (ResultMap)resultMaps.get(resultSetCount);
            this.handleResultSet(rsw, resultMap, multipleResults, (ResultMapping)null);
            rsw = this.getNextResultSet(stmt);
            this.cleanUpAfterHandlingResultSet();
            ++resultSetCount;
        }

        String[] resultSets = this.mappedStatement.getResultSets();
        if (resultSets != null) {
            while(rsw != null && resultSetCount < resultSets.length) {
                ResultMapping parentMapping = (ResultMapping)this.nextResultMaps.get(resultSets[resultSetCount]);
                if (parentMapping != null) {
                    String nestedResultMapId = parentMapping.getNestedResultMapId();
                    ResultMap resultMap = this.configuration.getResultMap(nestedResultMapId);
                    this.handleResultSet(rsw, resultMap, (List)null, parentMapping);
                }

                rsw = this.getNextResultSet(stmt);
                this.cleanUpAfterHandlingResultSet();
                ++resultSetCount;
            }
        }

        return this.collapseSingleResultList(multipleResults);
    }
}
```

## 总结
本篇文章，根据了一个查询语句执行的大致的流程：

- MapperProxy：Mapper接口代理类，相关数据是提前初始化的
- MapperMethod：具体接口方法的执行
- SqlSession:SQL入口
- Executor：具体执行逻辑入口
- StatementHandler：对Statement的封装
- ResultHandler：结果获取与处理转换