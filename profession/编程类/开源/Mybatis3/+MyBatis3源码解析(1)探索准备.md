# MyBatis3源码解析（1）探索准备
***
这是我参与2022首次更文挑战的第6天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7052884569032392740)

## 简介
本篇文章将使用原生的JDBC方式操作数据库，然后在使用Mybatis提供的方式操作数据库，通过对比两部分的操作，大致得到Mybatis所做的主要工作，为接下来的源码解析做准备

## 示例代码
完整的工程已放到GitHub上：https://github.com/lw1243925457/MybatisDemo/tree/master/example

### 原生的JDBC操作数据库
我们简单是写一个连接数据库、创建表、插入、查询、关闭的示例，如下所示：

```java
public class JdbcTest {

    @Test
    public void test() {
        Connection conn = null;
        Statement stmt = null;
        try {
            // STEP 1: 连接数据库
            // JDBC driver name and database URL
            String JDBC_DRIVER = "org.h2.Driver";
            Class.forName(JDBC_DRIVER);

            System.out.println("Connecting to database...");
            String DB_URL = "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1";
            //  Database credentials
            String USER = "sa";
            String PASS = "";
            conn = DriverManager.getConnection(DB_URL, USER, PASS);

            //STEP 2: 设置参数，查询
            System.out.println("Creating table in given database...");
            stmt = conn.createStatement();
            String sql =  "CREATE TABLE   REGISTRATION " +
                    "(id INTEGER not NULL, " +
                    " first VARCHAR(255), " +
                    " last VARCHAR(255), " +
                    " age INTEGER, " +
                    " PRIMARY KEY ( id ))";
            stmt.executeUpdate(sql);
            System.out.println("Created table in given database...");

            //STEP 4: 处理查询结果
            String insertSql = "insert into REGISTRATION (id, first, last, age) values (?, ?, ?, ?)";
            PreparedStatement insertPreparedStatement = conn.prepareStatement(insertSql);
            insertPreparedStatement.setInt(1, 1);
            insertPreparedStatement.setString(2, "1");
            insertPreparedStatement.setString(3, "1");
            insertPreparedStatement.setInt(4, 1);
            insertPreparedStatement.executeUpdate();

            String querySql = "select * from REGISTRATION";
            ResultSet resultSet = stmt.executeQuery(querySql);
            ResultSetMetaData metaData = resultSet.getMetaData();
            while (resultSet.next()) {
                Registration registration = new Registration();
                registration.setId(resultSet.getInt(metaData.getColumnName(1)));
                registration.setFirst(resultSet.getString(metaData.getColumnName(2)));
                registration.setLast(resultSet.getString(metaData.getColumnName(3)));
                registration.setAge(resultSet.getInt(metaData.getColumnName(4)));
                System.out.println(registration);
            }

            // STEP 4: 关闭连接
            stmt.close();
            conn.close();
        } catch(Exception se) {
	    ......
        } //end try
        System.out.println("Goodbye!");
    }

    @Data
    static class Registration {
        private int id;
        private String first;
        private String last;
        private int age;
    }
}
```

如上代码所示，核心的步骤如下：

- 1.连接数据库
- 2.查询，设置参数
- 3.得到查询结果，转成我们想要的对象
- 4.关闭连接

1和4看着还好，但2和3，如果要硬是这样写的话，那代码可以感受很大的重复性，写很多的这样的代码就是一种折磨吧，哈哈

### MyBatis操作数据库示例
下面我们来看看Mybatis3是如果操作的，示例代码如下：

为了简单，没有采用配置文件等方式，我们直接使用java代码的方式进行简单的配置

```java
public class MybatisTest {

    @Test
    public void test() {
	// 得到数据库链接
        try(SqlSession session = buildSqlSessionFactory().openSession()) {
            PersonMapper personMapper = session.getMapper(PersonMapper.class);
	    // 创建表
            personMapper.createTable();
	    // 插入数据
            personMapper.save(Person.builder().id(1L).name("1").build());
	    // 查询数据
            Person person = personMapper.getPersonById(1);
            System.out.println(person);
        }
    }

    // 构建数据库的连接信息和配置Mapper
    public static SqlSessionFactory buildSqlSessionFactory() {
        String JDBC_DRIVER = "org.h2.Driver";
        String DB_URL = "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1";
        String USER = "sa";
        String PASS = "";
        DataSource dataSource = new PooledDataSource(JDBC_DRIVER, DB_URL, USER, PASS);
        Environment environment = new Environment("Development", new JdbcTransactionFactory(), dataSource);
        Configuration configuration = new Configuration(environment);
        configuration.addMapper(PersonMapper.class);
        SqlSessionFactoryBuilder builder = new SqlSessionFactoryBuilder();
        return builder.build(configuration);
    }
}

public interface PersonMapper {

    @Insert("create table person(id int not null, name varchar(255))")
    Integer createTable();

    @Insert("Insert into person(id, name) values (#{id}, #{name})")
    Integer save(Person person);

    @Select("Select id, name from Person where id=#{id}")
    @Results(value = {
            @Result(property = "id", column = "id"),
            @Result(property="name", column = "name"),
    })
    Person getPersonById(Integer personId);
}
```

如上，我们可以看到代码量相比直接操作数据库少了很多，而且可读性也比直接操作数据库好

最大的简化点在：

1. 自动构建设置了查询相关的参数，语义清晰，如示例中查询表明根据Id查询

2. 自动映射转换结果成我们定义的业务对象，大大简化了操作

## 总结
从上面的示例代码中，我们大致感受到了MyBatis所做的主要工作，所以我们解析的源码分析工作会从下面的方面展开：

- 1.Mybatis的数据库连接
- 2.Mybatis的查询
- 3.Mybatis的结果映射
