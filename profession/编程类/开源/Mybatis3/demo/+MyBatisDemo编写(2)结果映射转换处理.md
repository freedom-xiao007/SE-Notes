# MyBatis Demo 编写（2）结果映射转换处理
***

这是我参与2022首次更文挑战的第17天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7052884569032392740)

## 简介
在上篇中，我们完成了MyBatis一部分功能的搭建，已经能通过Mapper接口类的编写，自动执行相关的语句了，接下来完善结果处理部分

## 最终效果展示
修改下我们的Mapper

```java
public interface PersonMapper {

    @Select("select * from person")
    List<Person> list();

    @Insert("insert into person (id, name) values ('1', '1')")
    void save();
}
```

测试代码如下：

```java
public class SelfMybatisTest {

    @Test
    public void test() {
        try(SelfSqlSession session = buildSqlSessionFactory()) {
            PersonMapper personMapper = session.getMapper(PersonMapper.class);
            personMapper.save();
            List<Person> personList = personMapper.list();
            for (Object person: personList) {
                System.out.println(person.toString());
            }
        }
    }

    public static SelfSqlSession buildSqlSessionFactory() {
        String JDBC_DRIVER = "org.h2.Driver";
        String DB_URL = "jdbc:h2:file:./testDb";
        String USER = "sa";
        String PASS = "";

        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(DB_URL);
        config.setUsername(USER);
        config.setPassword(PASS);
        config.setDriverClassName(JDBC_DRIVER);
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        DataSource dataSource = new HikariDataSource(config);

        SelfConfiguration configuration = new SelfConfiguration(dataSource);
        configuration.addMapper(PersonMapper.class);
        return new SelfSqlSession(configuration);
    }
}
```

输出如下：

```text
add sql source: mapper.mapper.PersonMapper.list
add sql source: mapper.mapper.PersonMapper.save
executor
executor
Person(id=1, name=1)
```

成功的返回我们自定义的Person对象，这个Demo已经有一点样子，算是达成了目标

下面是实现的相关细节

## Demo编写
完整的工程已放到GitHub上：https://github.com/lw1243925457/MybatisDemo/tree/master/

本篇文章的代码对应的Tag是: V2

### 思路梳理
要实现SQL查询结果到Java对象的转换，我们需要下面的东西：

- 1.返回的Java对象信息
- 2.对应的SQL表字段信息
- 3.SQL字段值到Java对象字段的转换处理
- 4.读取SQL结果，转换成Java对象

### 1.返回的Java对象信息
我们需要知道当前接口方法返回的Java对象信息，方便后面的读取SQL查询结果，转换成Java对象

借鉴MyBatis，我们定义下面一个类，来保存接口方法返回的对象和SQL查询结果字段与Java对象字段的TypeHandler处理器

```java
@Builder
@Data
public class ResultMap {

    private Object returnType;
    private Map<String,TypeHandler> typeHandlerMaps;
}
```

### 2.对应的SQL表字段信息
在以前的MyBatis源码解析中，我们大致知道获取TypeHandler是根据JavaType和JdbcType，我们就需要知道数据库表中各个字段的类型，方便后面去匹配对应的TypeHandler

我们在程序初始化的时候，读取数据库中所有的表，保存下其各个字段对应的jdbcType

可能不同表中有相关的字段，但是不同的类型，所以第一层是表名，第二层是字段名称，最后对应其jdbcType

代码如下：

```java
public class SelfConfiguration {

    /**
     * 读取数据库中的所有表
     * 获取其字段对应的类型
     * @throws SQLException e
     */
    private void initJdbcTypeCache() throws SQLException {
        try (Connection conn = dataSource.getConnection()){
            final DatabaseMetaData dbMetaData = conn.getMetaData();
            ResultSet tableNameRes = dbMetaData.getTables(conn.getCatalog(),null, null,new String[] { "TABLE" });
            final List<String> tableNames = new ArrayList<>(tableNameRes.getFetchSize());
            while (tableNameRes.next()) {
                tableNames.add(tableNameRes.getString("TABLE_NAME"));
            }

            for (String tableName : tableNames) {
                try {
                    String sql = "select * from " + tableName;
                    PreparedStatement ps = conn.prepareStatement(sql);
                    ResultSet rs = ps.executeQuery();
                    ResultSetMetaData meta = rs.getMetaData();
                    int columnCount = meta.getColumnCount();
                    Map<String, Integer> jdbcTypeMap = new HashMap<>(columnCount);
                    for (int i = 1; i < columnCount + 1; i++) {
                        jdbcTypeMap.put(meta.getColumnName(i).toLowerCase(), meta.getColumnType(i));
                    }
                    jdbcTypeCache.put(tableName.toLowerCase(), jdbcTypeMap);
                } catch (Exception ignored) {
                }
            }
        }
    }
}
```

### 3.SQL字段值到Java对象字段的转换处理
接下来我们要定义JavaType与jdbcType相互转换的TypeHandler

简化点，我们内置定义String和Long类型的处理，并在初始化的时候进行注册（还没涉及到参数转换处理，所以暂时定义jdbcType到JavaType的处理）

TypeHandler代码如下：

```java
public interface TypeHandler {

    Object getResult(ResultSet res, String cluName) throws SQLException;
}

public class StringTypeHandler implements TypeHandler {

    private static final StringTypeHandler instance = new StringTypeHandler();

    public static StringTypeHandler getInstance() {
        return instance;
    }

    @Override
    public Object getResult(ResultSet res, String cluName) throws SQLException {
        return res.getString(cluName);
    }
}

public class LongTypeHandler implements TypeHandler {

    private static LongTypeHandler instance = new LongTypeHandler();

    public static LongTypeHandler getInstance() {
        return instance;
    }

    @Override
    public Object getResult(ResultSet res, String cluName) throws SQLException {
        return res.getLong(cluName);
    }
}
```

默认初始化注册：

```java
public class SelfConfiguration {

    private void initTypeHandlers() {
        final Map<Integer, TypeHandler> varchar = new HashMap<>();
        varchar.put(JDBCType.VARCHAR.getVendorTypeNumber(), StringTypeHandler.getInstance());
        typeHandlerMap.put(String.class, varchar);

        final Map<Integer, TypeHandler> intType = new HashMap<>();
        intType.put(JDBCType.INTEGER.getVendorTypeNumber(), LongTypeHandler.getInstance());
        typeHandlerMap.put(Long.class, intType);
    }
}
```

接着重要的一步是构建接口函数方法返回的结果处理了，具体的细节如下，关键的都进行了相关的注释：

```java
public class SelfConfiguration {

    private final DataSource dataSource;
    private final Map<String, SqlSource> sqlCache = new HashMap<>();
    private final Map<String, ResultMap> resultMapCache = new HashMap<>();
    private final Map<String, Map<String, Integer>> jdbcTypeCache = new HashMap<>();
    private final Map<Class<?>, Map<Integer, TypeHandler>> typeHandlerMap = new HashMap<>();

        /**
     * Mapper添加
     * 方法路径作为唯一的id
     * 保存接口方法的SQL类型和方法
     * 保存接口方法返回类型
     * @param mapperClass mapper
     */
    public void addMapper(Class<?> mapperClass) {
        final String classPath = mapperClass.getPackageName();
        final String className = mapperClass.getName();
        for (Method method: mapperClass.getMethods()) {
            final String id = StringUtils.joinWith("." ,classPath, className, method.getName());
            for (Annotation annotation: method.getAnnotations()) {
                if (annotation instanceof Select) {
                    addSqlSource(id, ((Select) annotation).value(), SqlType.SELECT);
                    continue;
                }
                if (annotation instanceof Insert) {
                    addSqlSource(id, ((Insert) annotation).value(), SqlType.INSERT);
                }
            }

            // 构建接口函数方法返回值处理
            addResultMap(id, method);
        }

    }

    /**
     * 构建接口函数方法返回值处理
     * @param id 接口函数 id
     * @param method 接口函数方法
     */
    private void addResultMap(String id, Method method) {
        // 空直接发返回
        if (method.getReturnType().getName().equals("void")) {
            return;
        }

        // 获取返回对象类型
        // 这里需要特殊处理下，如果是List的话，需要特殊处理得到List里面的对象
        Type type = method.getGenericReturnType();
        Type returnType;
        if (type instanceof ParameterizedType) {
            returnType = ((ParameterizedType) type).getActualTypeArguments()[0];
        } else {
            returnType = method.getReturnType();
        }
        // 接口方法id作为key，值为 接口方法返回对象类型和其中每个字段对应处理的TypeHandler映射
        resultMapCache.put(id, ResultMap.builder()
                .returnType(returnType)
                .typeHandlerMaps(buildTypeHandlerMaps((Class<?>) returnType))
                .build());
    }

    /**
     * 构建实体类的每个字段对应处理的TypeHandler映射
     * @param returnType 接口函数返回对象类型
     * @return TypeHandler映射
     */
    private Map<String, TypeHandler> buildTypeHandlerMaps(Class<?> returnType) {
        // 这里默认取类名的小写为对应的数据库表名，当然也可以使用@TableName之类的注解
        final String tableName = StringUtils.substringAfterLast(returnType.getTypeName(), ".").toLowerCase();
        final Map<String, TypeHandler> typeHandler = new HashMap<>(returnType.getDeclaredFields().length);
        for (Field field : returnType.getDeclaredFields()) {
            final String javaType = field.getType().getName();
            final String name = field.getName();
            final Integer jdbcType = jdbcTypeCache.get(tableName).get(name);
            // 根据JavaType和jdbcType得到对应的TypeHandler
            typeHandler.put(javaType, typeHandlerMap.get(field.getType()).get(jdbcType));
        }
        return typeHandler;
    }
}
```

### 4.读取SQL结果，转换成Java对象
接下来就是SQL查询结果的处理了，主要是根据在初始化阶段构建好的针对每个返回类型ResultMap

- 根据ResultMap中的返回对象类型，生成对象实例
- 根据ResultMap中的TypeHandler映射，得到各个字段对应的TypeHandler，得到处理结果
- 反射调用对象的Set方法

代码如下：

```java
public class ResultHandler {

    public List<Object> parse(String id, ResultSet res, SelfConfiguration config) throws SQLException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
        if (res == null) {
            return null;
        }

        // 根据接口函数Id得到初始化时国家队ResultMap
        ResultMap resultMap = config.getResultType(id);

        final List<Object> list = new ArrayList<>(res.getFetchSize());
        while (res.next()) {
            // 根据接口函数返回类型，生成一个实例
            Class<?> returnType = (Class<?>) resultMap.getReturnType();
            Object val = returnType.getDeclaredConstructor().newInstance();
            
            for (Field field: returnType.getDeclaredFields()) {
                final String name = field.getName();
                // 根据返回对象的字段类型，得到对应的TypeHandler,调用TypeHandler处理得到结果
                TypeHandler typeHandler = resultMap.getTypeHandlerMaps().get(field.getType().getName());
                Object value = typeHandler.getResult(res, name);
                // 调用对象的Set方法
                String methodEnd = name.substring(0, 1).toUpperCase() + name.substring(1);
                Method setMethod = val.getClass().getDeclaredMethod("set" + methodEnd, field.getType());
                setMethod.invoke(val, value);
            }
            list.add(val);
        }
        return list;
    }
}
```

## 总结
本篇中完善了Demo中对查询结果集的自动处理转换部分，完成后，核心的功能就算已经完成了，基本达到了目标