# MyBatis3源码解析(3)数据库连接
***

## 简介
基于上篇的示例感受，下面我们探索下MyBatis连接数据库的细节是如果实现的，在日常使用中是如何能和Druid数据库连接池等配合起来的

## 源码解析
基于上篇的示例代码：

```java
public class MybatisTest {

    @Test
    public void test() {
        try(SqlSession session = buildSqlSessionFactory().openSession()) {
            PersonMapper personMapper = session.getMapper(PersonMapper.class);
            personMapper.createTable();
            personMapper.save(Person.builder().id(1L).name("1").build());
            Person person = personMapper.getPersonById(1);
            System.out.println(person);
        }
    }

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
```

当前我们想找的是与数据库建立连接的部分

通过阅读书籍：《MyBatis3源码深度解析》，我们大概知道执行是在Executor中，我们跟踪相关的代码

经过不懈的努力跟踪，得到下面的关键代码：

找到了在执行语句中，在Executor中获取连接的关键代码

```java
public class SimpleExecutor extends BaseExecutor {
    private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
	// 获取数据库连接
        Connection connection = this.getConnection(statementLog);
        Statement stmt = handler.prepare(connection, this.transaction.getTimeout());
        handler.parameterize(stmt);
        return stmt;
    }
}
```

继续跟踪，来到我们示例中定义的事务管理器：JdbcTransactionFactory

```java
public class JdbcTransaction implements Transaction {
    public Connection getConnection() throws SQLException {
        if (this.connection == null) {
            this.openConnection();
        }

        return this.connection;
    }

    protected void openConnection() throws SQLException {
        if (log.isDebugEnabled()) {
            log.debug("Opening JDBC Connection");
        }

        // 从DataSource中获取
        this.connection = this.dataSource.getConnection();
        if (this.level != null) {
            this.connection.setTransactionIsolation(this.level.getLevel());
        }

        this.setDesiredAutoCommit(this.autoCommit);
    }
}
```

上面可以看到是从DataSource中获取的，来到我们定义的：PooledDataSource

```java
public class PooledDataSource implements DataSource {
    public Connection getConnection() throws SQLException {
        return this.popConnection(this.dataSource.getUsername(), this.dataSource.getPassword()).getProxyConnection();
    }

    private PooledConnection popConnection(String username, String password) throws SQLException {
        while(conn == null) {
            synchronized(this.state) {
                PoolState var10000;
                if (!this.state.idleConnections.isEmpty()) {
		    ......
                } else if (this.state.activeConnections.size() < this.poolMaximumActiveConnections) {
		    // 获取数据库连接池连接
                    conn = new PooledConnection(this.dataSource.getConnection(), this);
                    if (log.isDebugEnabled()) {
                        log.debug("Created connection " + conn.getRealHashCode() + ".");
                    }
                } else {
		    ......
                }
                if (conn != null) {
		    ......
                }
            }
        }
	......
    }
}
```

我们继续跟下去：

```java
public class UnpooledDataSource implements DataSource {
    public Connection getConnection() throws SQLException {
        return this.doGetConnection(this.username, this.password);
    }

    private Connection doGetConnection(String username, String password) throws SQLException {
        Properties props = new Properties();
        if (this.driverProperties != null) {
            props.putAll(this.driverProperties);
        }

        if (username != null) {
            props.setProperty("user", username);
        }

        if (password != null) {
            props.setProperty("password", password);
        }

        return this.doGetConnection(props);
    }

    private Connection doGetConnection(Properties properties) throws SQLException {
        this.initializeDriver();
	// 看到了熟悉的原生的获取数据库连接的方式
        Connection connection = DriverManager.getConnection(this.url, properties);
        this.configureConnection(connection);
        return connection;
    }
}
```

到这里我们找到了源码中如何获取数据库连接的关键代码

其实就是对于原生数据库操作的封装

## 调用栈回顾
我们回过头来看看整个过程中的类调用栈：

- MybatisTest：我们的测试代码
- MapperProxy：MyBatis的Mapper的代理类
- MapperMethod：SQL执行相关
- DefaultSqlSession
- CachingExecutor
- BaseExecutor
- SimpleExecutor
- JdbcTransation
- PooledDataSource
- UnpooledDataSource