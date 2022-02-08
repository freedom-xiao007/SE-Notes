# MyBatis3源码解析(5)查询结果处理
***
这是我参与2022首次更文挑战的第10天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7052884569032392740)

## 简介
上篇中解析了MyBatis3中参数是如何传递处理的，本篇接着看看在获取到查询结果后，MyBatis3是如何将SQL查询结果与我们接口函数定义的返回结果对应的

## 源码
### 获取结果后处理的入口
在：[MyBatis3源码解析(3)查询语句执行](https://juejin.cn/post/7061427063793647647/)中，我们大致知道了查询结果处理的想代码位置，如下：

```java
public class PreparedStatementHandler extends BaseStatementHandler {
  @Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    // 前面的语句已经执行，处理结果返回
    return resultSetHandler.handleResultSets(ps);
  }
}
```

### SQL结果处理
下面是具体的处理逻辑，先获取结果集和Mapper中定义的返回结果的相关信息，然后对应进行处理

```java
public class DefaultResultSetHandler implements ResultSetHandler {
  @Override
  public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

    final List<Object> multipleResults = new ArrayList<>();

    int resultSetCount = 0;
    // 获取结果集，并用自定义包装起来
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    // 获取ResultMap对象，一般只有一个，其实就是需要返回的类信息之类的
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    while (rsw != null && resultMapCount > resultSetCount) {
      ResultMap resultMap = resultMaps.get(resultSetCount);
      // 在这里会循环获取SQL结果集并进行转换
      handleResultSet(rsw, resultMap, multipleResults, null);
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }

    // 这个不太请求，资料上说一般不会指定这个，暂时放过
    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
      while (rsw != null && resultSetCount < resultSets.length) {
        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
          String nestedResultMapId = parentMapping.getNestedResultMapId();
          ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
          handleResultSet(rsw, resultMap, null, parentMapping);
        }
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
      }
    }

    return collapseSingleResultList(multipleResults);
  }

  private ResultSetWrapper getFirstResultSet(Statement stmt) throws SQLException {
    ResultSet rs = stmt.getResultSet();
    while (rs == null) {
      // move forward to get the first resultset in case the driver
      // doesn't return the resultset as the first result (HSQLDB 2.1)
      if (stmt.getMoreResults()) {
        rs = stmt.getResultSet();
      } else {
        if (stmt.getUpdateCount() == -1) {
          // no more results. Must be no resultset
          break;
        }
      }
    }
    return rs != null ? new ResultSetWrapper(rs, configuration) : null;
  }
}
```

我们一直跟下去，中间步骤不是太关键，下发是SQL Result遍历和结果转换的相关代码

```java
public class DefaultResultSetHandler implements ResultSetHandler {
    private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
      throws SQLException {
    DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
    ResultSet resultSet = rsw.getResultSet();
    skipRows(resultSet, rowBounds);
    // SQLresult遍历
    while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
      ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
      // 将单条SQL结果转成对象
      Object rowValue = getRowValue(rsw, discriminatedResultMap, null);
      // 存储
      storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
    }
   }

   private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
    // 根据返回的类型生成对象
    Object rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
    if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
      final MetaObject metaObject = configuration.newMetaObject(rowValue);
      boolean foundValues = this.useConstructorMappings;
      if (shouldApplyAutomaticMappings(resultMap, false)) {
        foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
      }
      // 结果转换
      foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
      foundValues = lazyLoader.size() > 0 || foundValues;
      rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
    }
    return rowValue;
  }

  
  private boolean applyPropertyMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
    final List<String> mappedColumnNames = rsw.getMappedColumnNames(resultMap, columnPrefix);
    boolean foundValues = false;
    // 结果相关的属性
    final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
    // 遍历，结果对应对应的值
    for (ResultMapping propertyMapping : propertyMappings) {
      // 列名称
      String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
      if (propertyMapping.getNestedResultMapId() != null) {
        // the user added a column attribute to a nested result map, ignore it
        column = null;
      }
      if (propertyMapping.isCompositeResult()
          || (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH)))
          || propertyMapping.getResultSet() != null) {
        // 转换，获取值
        Object value = getPropertyMappingValue(rsw.getResultSet(), metaObject, propertyMapping, lazyLoader, columnPrefix);
        // issue #541 make property optional
        final String property = propertyMapping.getProperty();
        if (property == null) {
          continue;
        } else if (value == DEFERRED) {
          foundValues = true;
          continue;
        }
        if (value != null) {
          foundValues = true;
        }
        if (value != null || (configuration.isCallSettersOnNulls() && !metaObject.getSetterType(property).isPrimitive())) {
	  // 更新结果
          // gcode issue #377, call setter on nulls (value is not 'found')
          metaObject.setValue(property, value);
        }
      }
    }
    return foundValues;
  }

  private Object getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
    if (propertyMapping.getNestedQueryId() != null) {
      return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);
    } else if (propertyMapping.getResultSet() != null) {
      addPendingChildRelation(rs, metaResultObject, propertyMapping);   // TODO is that OK?
      return DEFERRED;
    } else {
      // 将SQL值转成对应java对象值，MyBatis3已内容Int Str等转换器
      final TypeHandler<?> typeHandler = propertyMapping.getTypeHandler();
      final String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
      return typeHandler.getResult(rs, column);
    }
  }
}
```

在上面代码中，我们看到了：

- 循环遍历SQL结果：ResultSet
- 遍历SQL列，转成Java类型

MyBatis中已经内置很多的类型转换器，也可以自己自定义，比如内容Long如下：

```java
public class LongTypeHandler extends BaseTypeHandler<Long> {
  @Override
  public Long getNullableResult(ResultSet rs, String columnName)
      throws SQLException {
    long result = rs.getLong(columnName);
    return result == 0 && rs.wasNull() ? null : result;
  }
}
```

## 总结
本篇文章中探索的MyBatis中如何将查询结果转换成我们想要的Java对象

- SQL执行
- 遍历ResultSet
- 遍历数据库Column，根据列名称，获取对应的值，转成Java对象