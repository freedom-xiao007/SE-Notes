# MyBatis3源码解析(4)参数解析
***

这是我参与2022首次更文挑战的第9天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7052884569032392740)

## 简介
上篇文章中探索了查询语句的执行过程，下面我们接着来看看其中的查询参数的解析细节，是如何工作的

## 参数的解析
在日常的开发中，常见的参数有如下几种：

- 1.直接传入： func(Object param1, Object param2, ...)
- 2.放入Map中进行传入：func(Map<String, Object> param)
- 3.类传入： func(Object param)

上面的请求是如何对应的呢，下面让我们带着疑问跟着源码走一走

### 参数解析前的相关准备工作
在上篇中，在ParamNameResolver有获取参数列表的代码，大体上从names中遍历获取的，这里就涉及到names的初始化的相关代码，如下：

```java
public class ParamNameResolver {
    public static final String GENERIC_NAME_PREFIX = "param";
    private final boolean useActualParamName;
    private final SortedMap<Integer, String> names;
    private boolean hasParamAnnotation;

    // 初始化话的时候，将相关的name已初始化好
    public ParamNameResolver(Configuration config, Method method) {
        this.useActualParamName = config.isUseActualParamName();
        Class<?>[] paramTypes = method.getParameterTypes();
        Annotation[][] paramAnnotations = method.getParameterAnnotations();
        SortedMap<Integer, String> map = new TreeMap();
        int paramCount = paramAnnotations.length;

        for(int paramIndex = 0; paramIndex < paramCount; ++paramIndex) {
            if (!isSpecialParameter(paramTypes[paramIndex])) {
                String name = null;
                Annotation[] var9 = paramAnnotations[paramIndex];
                int var10 = var9.length;

                for(int var11 = 0; var11 < var10; ++var11) {
                    Annotation annotation = var9[var11];
                    if (annotation instanceof Param) {
                        this.hasParamAnnotation = true;
                        name = ((Param)annotation).value();
                        break;
                    }
                }

                if (name == null) {
                    if (this.useActualParamName) {
                        name = this.getActualParamName(method, paramIndex);
                    }

                    if (name == null) {
                        name = String.valueOf(map.size());
                    }
                }

                map.put(paramIndex, name);
            }
        }

        this.names = Collections.unmodifiableSortedMap(map);
    }
}
```

在上面的代码中，names在类构造函数中已经生成好了，后面获取值的时候直接用即可

而在ParamNameResolver的构造函数中，通过初步跟踪代码，是直接读取的接口函数参数获取得到的参数，也就是在情况3中传入类，是当做一个参数，后面这个类会一直传递下去

ParamNameResolver在MapperMethod中就已经初始化好了

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {
    private MapperProxy.MapperMethodInvoker cachedInvoker(Method method) throws Throwable {
        try {
            return (MapperProxy.MapperMethodInvoker)MapUtil.computeIfAbsent(this.methodCache, method, (m) -> {
                if (m.isDefault()) {
		     ......
                } else {
                    return new MapperProxy.PlainMethodInvoker(new MapperMethod(this.mapperInterface, method, this.sqlSession.getConfiguration()));
                }
            });
        } catch (RuntimeException var4) {
            Throwable cause = var4.getCause();
            throw (Throwable)(cause == null ? var4 : cause);
        }
    }
}

public class MapperMethod {
    public static class MethodSignature {
	......
        public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
	    ......
            this.paramNameResolver = new ParamNameResolver(configuration, method);
        }
    }
}
```

跟踪代码下来，首先进行相关的初始化工作，而后在进行参数的解析获取

### 参数值初步获取
初始化完成后，在代码语句执行前，会获取参数值列表，下面是具体的处理逻辑：

```java
public class ParamNameResolver {
    public Object getNamedParams(Object[] args) {
	// 获取参数的数量
        int paramCount = this.names.size();
        if (args != null && paramCount != 0) {
	    // 没有参数声明并且参数数量为1
            if (!this.hasParamAnnotation && paramCount == 1) {
                Object value = args[(Integer)this.names.firstKey()];
                return wrapToMapIfCollection(value, this.useActualParamName ? (String)this.names.get(0) : null);
            } else {
                Map<String, Object> param = new ParamMap();
                int i = 0;

                // 遍历name获得参数列表
                for(Iterator var5 = this.names.entrySet().iterator(); var5.hasNext(); ++i) {
                    Entry<Integer, String> entry = (Entry)var5.next();
                    param.put(entry.getValue(), args[(Integer)entry.getKey()]);
                    String genericParamName = "param" + (i + 1);
                    if (!this.names.containsValue(genericParamName)) {
                        param.put(genericParamName, args[(Integer)entry.getKey()]);
                    }
                }

                return param;
            }
        } else {
            return null;
        }
    }
}
```

在上面处理中，如果参数列表中唯一只有一个类参数，那这个参数也算是一个参数，会直接返回类，比如在示例中会直接返回：Person（id=1，name=1），后面获取参数值填充时，会使用类get方法获取值，这个在下面会接着分析

而且注意在param中，会存入两个东西，一个argx，一个是paramx，感觉是和${}和#{}有关，这个后面再分析，param在示例中会如下：

```text
#如果不加@Param注解
param:
- arg0: 1
- arg1: 1
- param0: 1
- param1: 1

#如果加@Param注解
param:
- id: 1
- name: 1
- param0: 1
- param1: 1
```

在这里就得到后面要用的参数了，这里需要注意了，如果是单个参数，那就是直接返回对应的值；如果是多个参数，那就会放到一个map中，这个map中的key是非常关键的，因为构造preStatement是根据名称从里面取值的，后面会有相关代码

### PreStatement的生成
咋上面得到参数后，并不是直接使用，而在在PreStatement生成的时候用于传入的，关键的代码如下：

```java
public class SimpleExecutor extends BaseExecutor {
    private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
        Connection connection = this.getConnection(statementLog);
        Statement stmt = handler.prepare(connection, this.transaction.getTimeout());
	// 这里会进行参数的传入
        handler.parameterize(stmt);
        return stmt;
    }
}

public class DefaultParameterHandler implements ParameterHandler {
    public void setParameters(PreparedStatement ps) {
        ErrorContext.instance().activity("setting parameters").object(this.mappedStatement.getParameterMap().getId());
        List<ParameterMapping> parameterMappings = this.boundSql.getParameterMappings();
        if (parameterMappings != null) {
            for(int i = 0; i < parameterMappings.size(); ++i) {
                ParameterMapping parameterMapping = (ParameterMapping)parameterMappings.get(i);
                if (parameterMapping.getMode() != ParameterMode.OUT) {
                    String propertyName = parameterMapping.getProperty();
                    Object value;
                    if (this.boundSql.hasAdditionalParameter(propertyName)) {
                        value = this.boundSql.getAdditionalParameter(propertyName);
                    } else if (this.parameterObject == null) {
                        value = null;
                    } else if (this.typeHandlerRegistry.hasTypeHandler(this.parameterObject.getClass())) {
                        value = this.parameterObject;
                    } else {
			// 上面的三个先不管，下面就是获取参数的具体的逻辑
			// 如果是类，会通过一些处理调用对应的get方法
			// 如果多个之间传递的参数，在上面会放入一个map，之间从map中获取即可
                        MetaObject metaObject = this.configuration.newMetaObject(this.parameterObject);
                        value = metaObject.getValue(propertyName);
                    }

                    // 和参数拦截器相关的，后面再解析，先放过
                    TypeHandler typeHandler = parameterMapping.getTypeHandler();
                    JdbcType jdbcType = parameterMapping.getJdbcType();
                    if (value == null && jdbcType == null) {
                        jdbcType = this.configuration.getJdbcTypeForNull();
                    }

                    try {
			// 设置相关的值
                        typeHandler.setParameter(ps, i + 1, value, jdbcType);
                    } catch (SQLException | TypeException var10) {
                        throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + var10, var10);
                    }
                }
            }
        }

    }
}
```

从上面大致代码可以看到，在正常情况下，参数的设置都是通过名称取获取的，之间传入或者单个传入的情况比较简单

- 多个参数的情况下，会将所有的参数放入map中，后面根据名称去获取
- 单个参数类的情况下，会调用类的get方法对应获取

那如果混合类型，比如下面的情况：

```java
public interface PersonMapper {
    @Select("Select id, name from Person where id=#{id} and name=#{person.name}")
    @Results(value = {
            @Result(property = "id", column = "id"),
            @Result(property="name", column = "name"),
    })
    Person getPersonByMul(@Param("person") Person person, @Param("id") Integer id);
}
```

我们一直根据下，从下面的代码中得到，如果是类，会递归去获取

```java
public class MetaObject {
    public Object getValue(String name) {
        PropertyTokenizer prop = new PropertyTokenizer(name);
        if (prop.hasNext()) {
            MetaObject metaValue = this.metaObjectForProperty(prop.getIndexedName());
	    // 如果是类，后面会再次调用getVale获取
            return metaValue == SystemMetaObject.NULL_META_OBJECT ? null : metaValue.getValue(prop.getChildren());
        } else {
            return this.objectWrapper.get(prop);
        }
    }
}
```

## 总结
通过本篇的探索，我们大致了解了MyBatis3的参数获取解析原理

- 1.在初始化阶段，获取接口函数的参数列表，初始化names，用于后面获取参数值
- 2.获取参数值，分单个或者多个参数的情况，而且根据参数的类型，有不同的处理方法
  - 单个参数：直接返回
  - 多个参数：放入Map中返回，后面根据key进行获取
  - 参数类型是类：后面调用类的get防护获取对应的值
  - 参数类型是Map：后面直接调用get方法
- 3.PreStatement生成，生成的时候对应的值从上面得到的参数值获取