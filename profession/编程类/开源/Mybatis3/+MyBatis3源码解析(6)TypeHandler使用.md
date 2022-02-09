# MyBatis3源码解析(6)TypeHandler使用
***

这是我参与2022首次更文挑战的第12天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7052884569032392740)

## 简介
在上几篇中，介绍了MyBatis3对参数和结果的解析转换，对于常规数据类型，默认的处理已经足够应付了，但日常开发中会有一些特殊的类型，就可以通过TypeHandler来进行处理

## 示例准备
本篇文中用于调试的测试代码请参考：[MyBatis3源码解析（1）探索准备](https://juejin.cn/post/7058354949209456653)

完整的工程已放到GitHub上：https://github.com/lw1243925457/MybatisDemo/tree/master/example

熟悉使用的可以跳过该部分，直接到源码解析部分

### 数据模型修改
我们先将原来的String类型变成String数组,存入数据库时，使用逗号进行分割，取出时转成String数组，即

- Java数据结构：

```java
@Builder
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Person {

    private Long id;
    private String[] name;
}
```

- 存入数据库，已Varchar的方式：1,2,3

### 自定义TypeHandler编写
我们修改完数据结构后，需要编写一个自定义的TypeHandler，如下：

```java
public class StringArrayTypeHandler extends BaseTypeHandler<String[]> {

    @Override
    public void setNonNullParameter(PreparedStatement preparedStatement, int i, String[] strings, JdbcType jdbcType)
            throws SQLException {
        preparedStatement.setString(i, StringUtils.join(strings, ","));
    }

    @Override
    public String[] getNullableResult(ResultSet resultSet, String s) throws SQLException {
        return convert(resultSet.getString(s));
    }

    @Override
    public String[] getNullableResult(ResultSet resultSet, int i) throws SQLException {
        return convert(resultSet.getString(i));
    }

    @Override
    public String[] getNullableResult(CallableStatement callableStatement, int i) throws SQLException {
        return convert(callableStatement.getString(i));
    }

    /**
     * 将查询值转换为数组
     *
     * @param value 查询值, String
     * @return 转换结果, String[]
     */
    private String[] convert(String value) {
        return StringUtils.isEmpty(value) ? new String[0] : value.split(",");
    }
}
```

### TypeHandler注册
将我们自定义的TypeHandler进行注册，MyBatis后将对应String【】和Varchar使用此TypeHandler

```java
public class MybatisTest {
    public static SqlSessionFactory buildSqlSessionFactory() {
	......
        Configuration configuration = new Configuration(environment);
	// 注册TypeHandler，注意需要在添加Mapper之前，因为添加Mapper操作的时候就需要TypeHandler了
        configuration.getTypeHandlerRegistry().register(String[].class, JdbcType.VARCHAR, StringArrayTypeHandler.class);
        configuration.addMapper(PersonMapper.class);
        SqlSessionFactoryBuilder builder = new SqlSessionFactoryBuilder();
        return builder.build(configuration);
    }
}
```

### 运行
测试结果如下：

```text
Connected to the target VM, address: '127.0.0.1:53557', transport: 'socket'
[Person(id=1, name=[1, 2]), Person(id=2, name=[1, 2])]
Disconnected from the target VM, address: '127.0.0.1:53557', transport: 'socket'
```

## 源码解析
### 参数解析设置
通过前面几篇文章的分析，我们得到设置参数的相关源码如下：

```java
public class DefaultParameterHandler implements ParameterHandler {
  @Override
  public void setParameters(PreparedStatement ps) {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
      for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping parameterMapping = parameterMappings.get(i);
        if (parameterMapping.getMode() != ParameterMode.OUT) {
          Object value;
          String propertyName = parameterMapping.getProperty();
          if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
            value = boundSql.getAdditionalParameter(propertyName);
          } else if (parameterObject == null) {
            value = null;
          } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            value = parameterObject;
          } else {
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            value = metaObject.getValue(propertyName);
          }
	  // 获取对应的TypeHandler
          TypeHandler typeHandler = parameterMapping.getTypeHandler();
          JdbcType jdbcType = parameterMapping.getJdbcType();
          if (value == null && jdbcType == null) {
            jdbcType = configuration.getJdbcTypeForNull();
          }
          try {
	    // 调动对应的方法，设置值
            typeHandler.setParameter(ps, i + 1, value, jdbcType);
          } catch (TypeException | SQLException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          }
        }
      }
    }
  }
}
```

设置值，函数如下：

```java
public abstract class BaseTypeHandler<T> extends TypeReference<T> implements TypeHandler<T> {
  @Override
  public void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
    if (parameter == null) {
      if (jdbcType == null) {
        throw new TypeException("JDBC requires that the JdbcType must be specified for all nullable parameters.");
      }
      try {
        ps.setNull(i, jdbcType.TYPE_CODE);
      } catch (SQLException e) {
        throw new TypeException("Error setting null for parameter #" + i + " with JdbcType " + jdbcType + " . "
              + "Try setting a different JdbcType for this parameter or a different jdbcTypeForNull configuration property. "
              + "Cause: " + e, e);
      }
    } else {
      try {
	// 调用我们自定义的TypeHandler函数
        setNonNullParameter(ps, i, parameter, jdbcType);
      } catch (Exception e) {
        throw new TypeException("Error setting non null for parameter #" + i + " with JdbcType " + jdbcType + " . "
              + "Try setting a different JdbcType for this parameter or a different configuration property. "
              + "Cause: " + e, e);
      }
    }
  }
}
```

如上所示，我们看到TypeHandler调用的大致流程

其中MyBatis3是根据Java Type 和 JDBC Type 来对应调用不同的TypeHandler的，在示例代码中我们也显式的注册了 String数组和Varchar

这里使用的是一个全局的，也可以单独在查询中使用，可以自行搜索

### 结果解析返回
通过之前的文章，我们知道代码如下：

```java
  private Object getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
    if (propertyMapping.getNestedQueryId() != null) {
      return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);
    } else if (propertyMapping.getResultSet() != null) {
      addPendingChildRelation(rs, metaResultObject, propertyMapping);   // TODO is that OK?
      return DEFERRED;
    } else {
      // 得到对应的TypeHandler，直接返回处理结果
      final TypeHandler<?> typeHandler = propertyMapping.getTypeHandler();
      final String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
      return typeHandler.getResult(rs, column);
    }
  }
```

GetResult函数如下，后面调用我们自定义的TypeHandler方法

```java
  @Override
  public T getResult(ResultSet rs, String columnName) throws SQLException {
    try {
      return getNullableResult(rs, columnName);
    } catch (Exception e) {
      throw new ResultMapException("Error attempting to get column '" + columnName + "' from result set.  Cause: " + e, e);
    }
  }
```

## 总结
本篇文章，展示了如何定义TypeHandler和使用TypeHandler，并展示了源码部分，TypeHandler是如何对应生效的，了解该部分，在日常开发中，使用TypeHandler也能做到心中有数