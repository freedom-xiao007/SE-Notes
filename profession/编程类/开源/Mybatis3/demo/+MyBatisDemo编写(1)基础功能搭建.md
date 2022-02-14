# MyBatis Demo 编写（1）基础功能搭建
***

这是我参与2022首次更文挑战的第16天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7052884569032392740)

## 简介
在Mybatis3的源码解析系列中，我们对其核心功能有了一定的了解，下面我们尝试简单写一下Demo，让其有简单的Mybatis的一些核心功能，本篇是基础功能的搭建

## Dome 编写
完整的工程已放到GitHub上：https://github.com/lw1243925457/MybatisDemo/tree/master/

本篇文章的代码对应的Tag是: V1

本篇的目标是完成MapperProxy和运行不带参数和无自定义返回类型的SQL

测试代码如下：

```java
public class SelfMybatisTest {

    @Test
    public void test() {
        try(SelfSqlSession session = buildSqlSessionFactory()) {
            PersonMapper personMapper = session.getMapper(PersonMapper.class);
            personMapper.createTable();
            personMapper.save();
            List<Object> personList = personMapper.list();
            for (Object person: personList) {
                System.out.println(person.toString());
            }
        }
    }

    public static SelfSqlSession buildSqlSessionFactory() {
        String JDBC_DRIVER = "org.h2.Driver";
        String DB_URL = "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1";
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
//        configuration.getTypeHandlerRegistry().register(String[].class, JdbcType.VARCHAR, StringArrayTypeHandler.class);
        configuration.addMapper(PersonMapper.class);
        return new SelfSqlSession(configuration);
    }
}
```

如上所示，我们集成其他DataSource，这里使用HikariCP，然后初始化Mapper，生成表、插入、查询

最终的结果如下：

初始化我们的PersonMapper，最后输出查询的结果

```text
add sql source: mapper.mapper.PersonMapper.list
add sql source: mapper.mapper.PersonMapper.save
add sql source: mapper.mapper.PersonMapper.createTable

executor
executor
executor

[1, 1]
```

基本达到了要求，下面开始讲解核心代码：

### Mapper初始化
我们的思路目前是：

- 1.SelfConfiguration 作为全局配置类，从中取得DataSource和Mapper相关的信息（目前是接口方法对应的SQL语句）
- 2.SelfConfiguration 构造函数保存DataSource
- 3.SelfConfiguration 添加Mapper时，保存其中方式对应的SQL语句信息到Map，方法的路径作为唯一标识

SelfConfiguration 相关的代码

```java
public class SelfConfiguration {

    private final DataSource dataSource;
    private final Map<String, SqlSource> sqlCache = new HashMap<>();

    public SelfConfiguration(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    /**
     * Mapper添加
     * 保存接口方法的SQL类型和方法
     * 方法路径作为唯一的id
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
        }
    }

    private void addSqlSource(final String id, final String sql, final SqlType selectType) {
        System.out.println("add sql source: " + id);
        final SqlSource sqlSource = SqlSource.builder()
                .type(selectType)
                .sql(sql)
                .build();
        sqlCache.put(id, sqlSource);
    }

    public DataSource getDataSource() {
        return dataSource;
    }

    public SqlSource getSqlSource(final String id) {
        if (sqlCache.containsKey(id)) {
            return sqlCache.get(id);
        }
        throw new RuntimeException("don't find mapper match: " + id);
    }
}
```

### MapperProxy生成
MapperProxy 从 SelfSqlSession 中进行获取，MapperProxy中调用Executor的方法即可，比较简单

SelfSqlSession 主要是返回MapperProxy

```java
public class SelfSqlSession implements Closeable {

    private final SelfConfiguration config;

    public SelfSqlSession(SelfConfiguration configuration) {
        this.config = configuration;
    }

    @Override
    public void close() {
    }

    public <T> T getMapper(Class<?> mapperClass) {
        final MapperProxy<T> proxy = new MapperProxy(config);
        return (T) Proxy.newProxyInstance(mapperClass.getClassLoader(), new Class[] {mapperClass}, proxy);
    }
}
```

MapperProxy 简单调用下 Executor

```java
public class MapperProxy<T> implements InvocationHandler {

    private final SelfConfiguration config;

    public MapperProxy(final SelfConfiguration configuration) {
        this.config = configuration;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        return Executor.executor(config, proxy, method, args);
    }
}
```

### SQL语句执行
主要逻辑如下：

- SQL的编排主要还是在Executor中
- 语句的解析和执行在StatementHandler中，返回返回结果
- 结果的解析在ResultHandler中

Executor 代码如下，获取数据库连接，StatementHandler执行，ResultHandler处理结果返回

```java
public class Executor {

    private static final StatementHandler statementHandler = new StatementHandler();
    private static final ResultHandler resultHandler = new ResultHandler();

    public static Object executor(SelfConfiguration config, Object proxy, Method method, Object[] args) throws SQLException {
        System.out.println("executor");
        try (Connection conn = config.getDataSource().getConnection()) {
            ResultSet resultSet = statementHandler.prepare(conn, proxy, method, args, config);
            return resultHandler.parse(resultSet);
        }
    }
}
```

StetementHandler主要是从Config从获取需要执行的SQL信息

目前比较简单，根据SQL类型，直接执行即可

```java
public class StatementHandler {

    public ResultSet prepare(Connection conn, Object proxy, Method method, Object[] args, SelfConfiguration config) throws SQLException {
        final String classPath = method.getDeclaringClass().getPackageName();
        final String className = method.getDeclaringClass().getName();
        final String methodName = method.getName();
        final String id = StringUtils.joinWith(".", classPath, className, methodName);
        final SqlSource sqlSource = config.getSqlSource(id);
        if (sqlSource.getType().equals(SqlType.SELECT)) {
            return select(conn, sqlSource);
        }
        if (sqlSource.getType().equals(SqlType.INSERT)) {
            return insert(conn, sqlSource);
        }
        throw new RuntimeException("don't support this sql type");
    }

    private ResultSet insert(Connection conn, SqlSource sqlSource) throws SQLException {
        final Statement statement = conn.createStatement();
        statement.execute(sqlSource.getSql());
        return null;
    }

    private ResultSet select(Connection conn, SqlSource sqlSource) throws SQLException {
        final Statement statement = conn.createStatement();
        return statement.executeQuery(sqlSource.getSql());
    }
}
```

ResultHandler 目前就是循环读取结果，然后返回

```java
public class ResultHandler {

    public List<Object> parse(ResultSet res) throws SQLException {
        if (res == null) {
            return null;
        }

        final List<Object> list = new ArrayList<>(res.getFetchSize());
        final ResultSetMetaData metaData = res.getMetaData();
        while (res.next()) {
            final int count =metaData.getColumnCount();
            final List<Object> val = new ArrayList<>(count);
            for (int i=1; i <= count; i++) {
                final String name = metaData.getColumnName(i);
                final Object value = res.getObject(name);
                val.add(value);
            }
            list.add(val);
        }
        return list;
    }
}
```

## 总结
本篇中搭建了MyBatis Demo的基础部分，除了结果返回有点不好看，其他有了点日常MyBatis的味道了，结果处理后面也会完善上去