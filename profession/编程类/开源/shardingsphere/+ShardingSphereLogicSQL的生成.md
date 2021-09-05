# ShardingSphere LogicSQL 的生成探索
***
## 简介
在上两篇文中，我们探索了SQLToken和真实SQL的生成的想关代码，本文继续来探索最开始的一个LogicSQL的生成，补全这一块拼图

## 源码探索
继续上面两篇的探索：

- [ShardingSphere 语句解析生成初探](https://juejin.cn/post/7003073129643769869)
- [ShardingSphere SQLToken 生成探索](https://juejin.cn/post/7003899320357355550)

其中生成真实的SQL有两个要素：

- 逻辑表名到真实表名的映射：这个在SQLToken生成的
- 拼接SQL语句时，对应的index位置，这个目前看来是在LogicSQL中就生成好了的

目前就还差index生成的东西，我们接下来就看看LogicSQL的相关代码：

### 寻找切入点
下面是LogicSQL的生成部分：

```java
    private ExecutionContext createExecutionContext() {
        LogicSQL logicSQL = createLogicSQL();
        SQLCheckEngine.check(logicSQL.getSqlStatementContext().getSqlStatement(), logicSQL.getParameters(), 
                metaDataContexts.getMetaData(connection.getSchemaName()).getRuleMetaData().getRules(), connection.getSchemaName(), metaDataContexts.getMetaDataMap(), null);
        ExecutionContext result = kernelProcessor.generateExecutionContext(logicSQL, metaDataContexts.getMetaData(connection.getSchemaName()), metaDataContexts.getProps());
        findGeneratedKey(result).ifPresent(generatedKey -> generatedValues.addAll(generatedKey.getGeneratedValues()));
        return result;
    }
    
    private LogicSQL createLogicSQL() {
        List<Object> parameters = new ArrayList<>(getParameters());
        SQLStatementContext<?> sqlStatementContext = SQLStatementContextFactory.newInstance(metaDataContexts.getMetaDataMap(), parameters, sqlStatement, connection.getSchemaName());
        return new LogicSQL(sqlStatementContext, sql, parameters);
    }
```

但通过debug看到相关的东西其实之前就生成好了：


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c845fff594d84e0794eca5a379f0ee72~tplv-k3u1fbpfcp-watermark.image)

其中的preparedStatement就有了相关的index信息，看来是在某一步就初始化好了的，我们找到对应的初始化语句，如下：

```java
# ShardingSpherePreparedStatement.java
    private ShardingSpherePreparedStatement(final ShardingSphereConnection connection, final String sql,
                                            final int resultSetType, final int resultSetConcurrency, final int resultSetHoldability, final boolean returnGeneratedKeys) throws SQLException {
        if (Strings.isNullOrEmpty(sql)) {
            throw new SQLException(SQLExceptionConstant.SQL_STRING_NULL_OR_EMPTY);
        }
        this.connection = connection;
        metaDataContexts = connection.getContextManager().getMetaDataContexts();
        this.sql = sql;
        statements = new ArrayList<>();
        parameterSets = new ArrayList<>();
        ShardingSphereSQLParserEngine sqlParserEngine = new ShardingSphereSQLParserEngine(
                DatabaseTypeRegistry.getTrunkDatabaseTypeName(metaDataContexts.getMetaData(connection.getSchemaName()).getResource().getDatabaseType()));
	// 这里进行生成的
        sqlStatement = sqlParserEngine.parse(sql, true);
        parameterMetaData = new ShardingSphereParameterMetaData(sqlStatement);
        statementOption = returnGeneratedKeys ? new StatementOption(true) : new StatementOption(resultSetType, resultSetConcurrency, resultSetHoldability);
        JDBCExecutor jdbcExecutor = new JDBCExecutor(metaDataContexts.getExecutorEngine(), connection.isHoldTransaction());
        driverJDBCExecutor = new DriverJDBCExecutor(connection.getSchemaName(), metaDataContexts, jdbcExecutor);
        rawExecutor = new RawExecutor(metaDataContexts.getExecutorEngine(), connection.isHoldTransaction(), metaDataContexts.getProps());
        // TODO Consider FederateRawExecutor
        federateExecutor = new FederateJDBCExecutor(connection.getSchemaName(), metaDataContexts.getOptimizeContextFactory(), metaDataContexts.getProps(), jdbcExecutor);
        batchPreparedStatementExecutor = new BatchPreparedStatementExecutor(metaDataContexts, jdbcExecutor, connection.getSchemaName());
        kernelProcessor = new KernelProcessor();
    }
```

我们再往前找找，看到是在OrderRepositoryImpl.java中进行触发的：

```java
# OrderRepositoryImpl.java
    @Override
    public Long insert(final Order order) throws SQLException {
        String sql = "INSERT INTO t_order (user_id, address_id, status) VALUES (?, ?, ?)";
        try (Connection connection = dataSource.getConnection();
             PreparedStatement preparedStatement = connection.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            preparedStatement.setInt(1, order.getUserId());
            preparedStatement.setLong(2, order.getAddressId());
            preparedStatement.setString(3, order.getStatus());
            preparedStatement.executeUpdate();
            try (ResultSet resultSet = preparedStatement.getGeneratedKeys()) {
                if (resultSet.next()) {
                    order.setOrderId(resultSet.getLong(1));
                }
            }
        }
        return order.getOrderId();
    }
```

那我们就继续探索：sqlStatement = sqlParserEngine.parse(sql, true);

我们一直跟着下去，来到一个SQL处理的相关类：MySQLStatementParser.java

其给人的第一个感觉是相当的复杂，我们跟着debug下去，看到进入到相关的insert处理的分支

```java
			case XA:
				enterOuterAlt(_localctx, 1);
				{
				setState(1246);
				_errHandler.sync(this);
				switch ( getInterpreter().adaptivePredict(_input,0,_ctx) ) {
				case 1:
					{
					setState(1146);
					select();
					}
					break;
				case 2:
					{
					setState(1147);
					insert();
					}
					break;

```

跟着下去，来到insert语句处理具体函数：

```java
# MySQLStatementParser.java
	public final InsertContext insert() throws RecognitionException {
		InsertContext _localctx = new InsertContext(_ctx, getState());
		enterRule(_localctx, 2, RULE_insert);
		int _la;
		try {
			enterOuterAlt(_localctx, 1);
			{
			setState(1258);
			// Insert相关处理
			match(INSERT);
			setState(1259);
			insertSpecification();
			setState(1261);
			_errHandler.sync(this);
			_la = _input.LA(1);
			if (_la==INTO) {
				{
				setState(1260);
				// into 相关处理
				match(INTO);
				}
			}

			setState(1263);
			// 表名相关处理
			tableName();
			setState(1265);
			_errHandler.sync(this);
			_la = _input.LA(1);
			if (_la==PARTITION) {
				{
				setState(1264);
				partitionNames();
				}
			}

			setState(1270);
			_errHandler.sync(this);
			switch ( getInterpreter().adaptivePredict(_input,6,_ctx) ) {
			case 1:
				{
				setState(1267);
				// values相关处理
				insertValuesClause();
				}
				break;
			case 2:
				{
				setState(1268);
				setAssignmentsClause();
				}
				break;
			case 3:
				{
				setState(1269);
				insertSelectClause();
				}
				break;
			}
			setState(1273);
			_errHandler.sync(this);
			_la = _input.LA(1);
			if (_la==ON) {
				{
				setState(1272);
				onDuplicateKeyClause();
				}
			}

			}
		}
		catch (RecognitionException re) {
			_localctx.exception = re;
			_errHandler.reportError(this, re);
			_errHandler.recover(this, re);
		}
		finally {
			exitRule();
		}
		return _localctx;
	}
```

在上面的函数中，我们大意看到几个比较关键的处理函数：

- Insert相关处理 : match(INSERT);
- into 相关处理 : match(INTO);
- 表名相关处理 : tableName();
- values相关处理 : insertValuesClause();

其规则跟下来有点复杂了，有循环和嵌套处理的，目前是梳理不清楚了

但其大意都是得到对应的开始和结束位置之类的，如下图：


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ac52f5a21f2d492886920699ba13e52f~tplv-k3u1fbpfcp-watermark.image)

最终得到结果如下：


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6fadbe3480fc463cb706d50dc62758c6~tplv-k3u1fbpfcp-watermark.image)

到结果后，相关的返回函数如下：

```java
@RequiredArgsConstructor
public final class SQLParserExecutor {
    
    private final String databaseType;
    
    /**
     * Parse SQL.
     * 
     * @param sql SQL to be parsed
     * @return parse tree
     */
    public ParseTree parse(final String sql) {
        ParseASTNode result = twoPhaseParse(sql);
        if (result.getRootNode() instanceof ErrorNode) {
            throw new SQLParsingException("Unsupported SQL of `%s`", sql);
        }
        return result.getRootNode();
    }
}
```

result.getRootNode() 如下：

```java
@RequiredArgsConstructor
public final class ParseASTNode implements ASTNode {
    
    private final ParseTree parseTree;
    
    /**
     * Get root node.
     * 
     * @return root node
     */
    public ParseTree getRootNode() {
        return parseTree.getChild(0);
    }
}
```

而result的结果如下图，getChild（0）就是得到上面我们Insert解析后得到的结果


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/90ef010f18ac49fa836afa9dd67c0433~tplv-k3u1fbpfcp-watermark.image)

## 总结
感觉看的迷迷糊糊的，很多地方目前还不能很好的理解

但我们起码通过本次的探索知道了真实SQL的关键路径：

- 通过原始的LogicSQL语句，经过的ShardingSphere的语法树解析，得到对应的各个部分的元数据，如开始和结束index
- 根据语法树解析结果，得到对应的SQLToken，其中包含了如分库分表中的逻辑表到真实表的映射等关键信息
- 根据SQLToken生成真实的SQL