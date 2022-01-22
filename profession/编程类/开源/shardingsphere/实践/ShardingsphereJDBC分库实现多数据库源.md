# ShardingSphere JDBC 分库实现多数据库源
***
这是我参与2022首次更文挑战的第2天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7052884569032392740)

## 简介
基于Shardingsphere JDBC 5.0.0版本，利用Sharding分库实现日常开始中的数据库多数据源使用需求，结合Spring Boot 和 Mybatis Plus

## 数据源需求说明
数据库初始语句如下：

```mysql
create database demo1;
create database demo2;

create table `demo1`.table1 (
    id int
);

create table `demo2`.table2 (
    id int
);

create table `demo1`.sharding_table (
    id int
);

create table `demo2`.sharding_table (
    id int
);

insert into `demo1`.sharding_table (id) values(1);
insert into `demo2`.sharding_table (id) values(1);
```

两个数据库，数据库1有表：table1、sharding_table

数据库2有表：table2、sharding_table

要求如下：

- 当访问表 table1 时，访问数据库 demo1
- 当访问表 table2 时，访问数据库 demo2
- 当访问表 sharding_table 时，根据自定义的传入参数，访问对应的数据，本篇文章，将要访问的数据源存入ThreadLocal中，获取后访问对应的数据源

## 关键代码示例
完整代码GitHub地址：https://github.com/lw1243925457/JAVA-000/tree/main/code/shardingsphere/shardingdb

### 定义数据源
配置ShardingSphere JDBC数据源，关键代码如下：

配置如下,定义了连个数据源，最后的rules是标识表table1到数据源db0访问，表table2到数据源db1访问

```yml
# shardingSphere 分库设置
shardingsphere:
  # 配置真实数据源
  datasources:
    # 数据库1
    db0:
      jdbcurl: ${DB1_URL:jdbc:mysql://127.0.0.1:3306/demo1?useUnicode=true&serverTimezone=UTC}
      username: ${DB1_USER:root}
      password: ${DB1_PASS:root}
    # 数据库2
    db1:
      jdbcurl: ${DB2_URL:jdbc:mysql://127.0.0.1:3306/demo2?useUnicode=true&serverTimezone=UTC}
      username: ${DB2_USER:root}
      password: ${DB2_PASS:root}
  rules:
    table1: db0
    table2: db1
```

如果使用ShardingSphere的yaml文件配置，暂时还没有找到如何使用环境变量的方式，不方便修改，所有使用Java代码直接进行配置

```java
@Slf4j
@Configuration
public class ShardingDataSourceMybatisPlusConfig extends MybatisPlusAutoConfiguration {

    private final MultipleDbConfig multipleDbConfig;

    @Primary
    @Bean("dataSource")
    public DataSource getDataSource() throws SQLException {
        // 配置真实数据源
        Map<String, MultipleDbConfig.DbSource> dbs = multipleDbConfig.getDatasources();
        Map<String, DataSource> dataSourceMap = new HashMap<>(dbs.size());
        for (String dbName: dbs.keySet()) {
            MultipleDbConfig.DbSource dbConfig = dbs.get(dbName);
            HikariDataSource dataSource = new HikariDataSource();
            dataSource.setDriverClassName("com.mysql.jdbc.Driver");
            dataSource.setJdbcUrl(dbConfig.getJdbcUrl());
            dataSource.setUsername(dbConfig.getUsername());
            dataSource.setPassword(dbConfig.getPassword());
            dataSourceMap.put(dbName, dataSource);
        }

        // 配置分片规则
        ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();

        // 遍历表的固定映射：表table1到数据源db0访问，表table2到数据源db1访问
        Map<String, String> rules = multipleDbConfig.getRules();
        for (final String table: rules.keySet()) {
            // 配置添加 t_order 表规则
            final String actualDataNodes = String.join(".", rules.get(table), table);
            shardingRuleConfig.getTables().add(new ShardingTableRuleConfiguration(table, actualDataNodes));
        }

        // 配置 sharding_table 表的访问，需要自定义实现分库和分表算法
        ShardingTableRuleConfiguration ShardingTableRuleConfiguration = new ShardingTableRuleConfiguration("sharding_table", "db${0..1}.sharding_table");
        shardingRuleConfig.setDefaultDatabaseShardingStrategy(new StandardShardingStrategyConfiguration("id", "customDbSharding"));
        shardingRuleConfig.setDefaultTableShardingStrategy(new StandardShardingStrategyConfiguration("id", "customTableSharding"));
        shardingRuleConfig.getTables().add(ShardingTableRuleConfiguration);

        // 配置分库算法
        Properties dbShardingAlgorithmProps = new Properties();
        dbShardingAlgorithmProps.setProperty("strategy", "standard");
        dbShardingAlgorithmProps.setProperty("algorithmClassName", "com.shardingsphere.shardingdb.config.CustomDbSharding");
        shardingRuleConfig.getShardingAlgorithms().put("customDbSharding", new ShardingSphereAlgorithmConfiguration("CLASS_BASED", dbShardingAlgorithmProps));

        // 配置分表算法
        Properties tableShardingAlgorithmProps = new Properties();
        tableShardingAlgorithmProps.setProperty("strategy", "standard");
        tableShardingAlgorithmProps.setProperty("algorithmClassName", "com.shardingsphere.shardingdb.config.CustomTableSharding");
        shardingRuleConfig.getShardingAlgorithms().put("customTableSharding", new ShardingSphereAlgorithmConfiguration("CLASS_BASED", tableShardingAlgorithmProps));

        // 开启Sql日志
        final Properties properties = new Properties();
        properties.setProperty("sql-show", "true");

        // 创建 ShardingSphereDataSource
        return ShardingSphereDataSourceFactory.createDataSource(dataSourceMap, Collections.singleton(shardingRuleConfig), properties);
    }

    @Override
    @Bean("sqlSessionFactory")
    public SqlSessionFactory sqlSessionFactory(@Qualifier("dataSource")DataSource dataSource) throws Exception {
        return super.sqlSessionFactory(getDataSource());
    }

    @Override
    @Bean("sqlSessionTemplate")
    public SqlSessionTemplate sqlSessionTemplate(@Qualifier("sqlSessionFactory")SqlSessionFactory sqlSessionFactory) {
        return super.sqlSessionTemplate(sqlSessionFactory);
    }
}
```

从代码上可以看出，大部分还是便于后面直接修复配置文件进行扩展的

自定义分库代码如下：主要是获取ThreadLocal中的数据源名称信息，然后返回给Shardingsphere，这样就能访问对应的数据源

示例中只是为了简单而使用这种直接的方式，也可以放入其他信息，自行根据需求转成对应的数据源

```java
public final class CustomDbSharding implements StandardShardingAlgorithm<Integer> {

    @Override
    public void init() {
    }

    @Override
    public String doSharding(final Collection<String> availableTargetNames, final PreciseShardingValue<Integer> shardingValue) {
        String dbName = ThreadLocalCache.threadLocal.get();
        for (String each : availableTargetNames) {
            if (each.equals(dbName)) {
                return each;
            }
        }
        return null;
    }

    @Override
    public Collection<String> doSharding(final Collection<String> availableTargetNames, final RangeShardingValue<Integer> shardingValue) {
        return availableTargetNames;
    }

    @Override
    public String getType() {
        return null;
    }
}
```

自定义分表，之类其实应该没有，但为了展示一个完整的，所以也弄了一个自定义分表，这里是直接返回即可

```java
public final class CustomTableSharding implements StandardShardingAlgorithm<Integer> {

    @Override
    public void init() {
    }

    @Override
    public String doSharding(final Collection<String> availableTargetNames, final PreciseShardingValue<Integer> shardingValue) {
        for (String each : availableTargetNames) {
            return each;
        }
        return null;
    }

    @Override
    public Collection<String> doSharding(final Collection<String> availableTargetNames, final RangeShardingValue<Integer> shardingValue) {
        return availableTargetNames;
    }

    @Override
    public String getType() {
        return null;
    }
}
```

### Entity、Mapper定义
简单的写写即可：

```java
@Data
@TableName("sharding_table")
public class ShardingTable {
    private Long id;
}

@Data
@TableName("table1")
public class Table1 {
    private Long id;
}

@Data
@TableName("table2")
public class Table2 {

    private Long id;
}

@Repository
public interface ShardingTableMapper extends BaseMapper<ShardingTable> {
}

@Repository
public interface Table1Mapper extends BaseMapper<Table1> {
}

@Repository
public interface Table2Mapper extends BaseMapper<Table2> {
}
```

### 测试验证
我们写了测试类，进行测试即可

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class ShardingDbTest {

    @Autowired
    private Table1Mapper table1Mapper;
    @Autowired
    private Table2Mapper table2Mapper;
    @Autowired
    private ShardingTableMapper shardingTableMapper;

    @Test
    public void test() {
        final List<Table1> l1 = table1Mapper.selectList(null);
        l1.forEach(System.out::println);

        final List<Table2> l2 = table2Mapper.selectList(null);
        l2.forEach(System.out::println);

        ThreadLocalCache.threadLocal.set("db1");
        System.out.println(shardingTableMapper.selectById(1L));

        ThreadLocalCache.threadLocal.set("db0");
        System.out.println(shardingTableMapper.selectById(1L));
    }
}
```

结果如下：

```text
Logic SQL: SELECT  id  FROM table1
SQLStatement: MySQLSelectStatement(limit=Optional.empty, lock=Optional.empty, window=Optional.empty)
Actual SQL: db0 ::: SELECT  id  FROM table1

Logic SQL: SELECT  id  FROM table2
SQLStatement: MySQLSelectStatement(limit=Optional.empty, lock=Optional.empty, window=Optional.empty)
Actual SQL: db1 ::: SELECT  id  FROM table2

Logic SQL: SELECT id FROM sharding_table WHERE id=? 
SQLStatement: MySQLSelectStatement(limit=Optional.empty, lock=Optional.empty, window=Optional.empty)
Actual SQL: db1 ::: SELECT id FROM sharding_table WHERE id=?  ::: [1]
ShardingTable(id=1)

Logic SQL: SELECT id FROM sharding_table WHERE id=? 
SQLStatement: MySQLSelectStatement(limit=Optional.empty, lock=Optional.empty, window=Optional.empty)
Actual SQL: db0 ::: SELECT id FROM sharding_table WHERE id=?  ::: [1]
```

可以看到访问四次，Actual SQL符合我们的预期

## 总结
展示了如何使用Shardingsphere JDBC实现多数据源访问，Shardingsphere JDBC如何实现自定义的分库和分表算法

## 参考链接
- [shardingsphere 使用 JAVA API](https://shardingsphere.apache.org/document/5.0.0/cn/user-manual/shardingsphere-jdbc/usage/sharding/java-api/)
- [shardingsphere YAML 配置 数据分片](https://shardingsphere.apache.org/document/5.0.0/cn/user-manual/shardingsphere-jdbc/configuration/yaml/sharding/)
- [ThreadLocal使用与原理](https://juejin.cn/post/6959333602748268575)
- [shardingsphere 分片算法](https://shardingsphere.apache.org/document/5.0.0/cn/user-manual/shardingsphere-jdbc/configuration/built-in-algorithm/sharding/)
- [How to configure a custom sharding strategy?](https://github.com/apache/shardingsphere/issues/1976)