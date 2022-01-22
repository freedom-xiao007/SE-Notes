# ShardingSphere JDBC 分库分表 读写分离 数据加密
***
## 简介
在上篇文章中，在本地搭建了运行环境，本地来体验下ShardingSphere JDBC的三个功能：分库分表、读写分离、数据加密

## 示例运行
首先把概念先捋一捋，参考下面的文档：

- [数据分片](https://shardingsphere.apache.org/document/current/cn/features/sharding/)
- [读写分离](https://shardingsphere.apache.org/document/current/cn/features/readwrite-splitting/)
- [数据加密](https://shardingsphere.apache.org/document/current/cn/features/encrypt/)

配置的参考说明也是要看一看的，参考下面的文档：

- [数据分片配置](https://shardingsphere.apache.org/document/current/cn/user-manual/shardingsphere-jdbc/configuration/yaml/sharding/)
- [读写分离配置](https://shardingsphere.apache.org/document/current/cn/user-manual/shardingsphere-jdbc/configuration/yaml/readwrite-splitting-/)
- [数据加密配置](https://shardingsphere.apache.org/document/current/cn/user-manual/shardingsphere-jdbc/configuration/yaml/encrypt/)

接下来就是运行示例了，简单点就运行官方源码中的示例：examples/shardingsphere-jdbc-example/sharding-example/sharding-spring-boot-mybatis-example

### 分库分表与读写分离
#### 1.数据库初始化
首先把相关的数据分片和读写分离所需要的表在数据库中建好

数据库简单使用docker起一个，用户名和密码都是root：

```shell script
docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root -d mysql:latest
```

运行下面的SQL语句建立相关的数据库：

```sql
CREATE SCHEMA IF NOT EXISTS demo_write_ds_0;
CREATE SCHEMA IF NOT EXISTS demo_write_ds_0_read_0;
CREATE SCHEMA IF NOT EXISTS demo_write_ds_0_read_1;
CREATE SCHEMA IF NOT EXISTS demo_write_ds_1;
CREATE SCHEMA IF NOT EXISTS demo_write_ds_1_read_0;
CREATE SCHEMA IF NOT EXISTS demo_write_ds_1_read_1;
```

#### 2.配置修改
先修改：examples/shardingsphere-jdbc-example/sharding-example/sharding-spring-boot-mybatis-example/src/main/resources/application.properties

```yaml
mybatis.config-location=classpath:META-INF/mybatis-config.xml

#spring.profiles.active=sharding-databases
#spring.profiles.active=sharding-tables
#spring.profiles.active=sharding-databases-tables
#spring.profiles.active=readwrite-splitting
spring.profiles.active=sharding-readwrite-splittin
```

将其配置文件修改为执行读写分离加分片

再修改：examples/shardingsphere-jdbc-example/sharding-example/sharding-spring-boot-mybatis-example/src/main/resources/application-sharding-readwrite-splitting.properties

简单的修改下对应的数据库密码，然后在最后添加下面一个配置，用于打印SQL语句：

```yaml
spring.shardingsphere.props.sql-show=true
```

#### 3.运行
跑起来后大致如下图所示：


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e8952a99ea594c2a9546eeb98c25de67~tplv-k3u1fbpfcp-watermark.image)

其中打印了插入语句和查询语句，示例如下：

```
---------------------------- Insert Data ----------------------------
Logic SQL: INSERT INTO t_order (user_id, address_id, status) VALUES (?, ?, ?); 
SQLStatement: MySQLInsertStatement(setAssignment=Optional.empty, onDuplicateKeyColumns=Optional.empty) 
Actual SQL: write-ds-1 ::: INSERT INTO t_order_0 (user_id, address_id, status, order_id) VALUES (?, ?, ?, ?); ::: [1, 1, INSERT_TEST, 636678694781825024] 
Logic SQL: INSERT INTO t_order_item (order_id, user_id, status) VALUES (?, ?, ?); 
SQLStatement: MySQLInsertStatement(setAssignment=Optional.empty, onDuplicateKeyColumns=Optional.empty) 
Actual SQL: write-ds-1 ::: INSERT INTO t_order_item_0 (order_id, user_id, status, order_item_id) VALUES (?, ?, ?, ?); ::: [636678694781825024, 1, INSERT_TEST, 636678695339667457] 
SQLStatement: MySQLInsertStatement(setAssignment=Optional.empty, onDuplicateKeyColumns=Optional.empty) 
Actual SQL: write-ds-0 ::: INSERT INTO t_order_0 (user_id, address_id, status, order_id) VALUES (?, ?, ?, ?); ::: [2, 2, INSERT_TEST, 636678695436136448] 
Logic SQL: INSERT INTO t_order_item (order_id, user_id, status) VALUES (?, ?, ?); 
```

插入的语句大致就如上面的，可以从Actual SQL看出，数据的生成插入确实是分片了的

但读写分离，就不太确定了：

```
---------------------------- Print Order Data -----------------------
[INFO ] 2021-08-23 21:33:50,057 --main-- [ShardingSphere-SQL] Logic SQL: SELECT * FROM t_order; 
[INFO ] 2021-08-23 21:33:50,057 --main-- [ShardingSphere-SQL] SQLStatement: MySQLSelectStatement(limit=Optional.empty, lock=Optional.empty, window=Optional.empty) 
[INFO ] 2021-08-23 21:33:50,057 --main-- [ShardingSphere-SQL] Actual SQL: write-ds-0 ::: SELECT * FROM t_order_0 ORDER BY order_id ASC ; 
[INFO ] 2021-08-23 21:33:50,057 --main-- [ShardingSphere-SQL] Actual SQL: write-ds-0 ::: SELECT * FROM t_order_1 ORDER BY order_id ASC ; 
[INFO ] 2021-08-23 21:33:50,057 --main-- [ShardingSphere-SQL] Actual SQL: write-ds-1 ::: SELECT * FROM t_order_0 ORDER BY order_id ASC ; 
[INFO ] 2021-08-23 21:33:50,057 --main-- [ShardingSphere-SQL] Actual SQL: write-ds-1 ::: SELECT * FROM t_order_1 ORDER BY order_id ASC ; 
```

读的话，从语句上来看，还是读的write，不是预想中的：write-ds-0-read-0,write-ds-0-read-1,write-ds-1-read-0,write-ds-1-read-1，其中一个，这个有点疑惑

后面自己又运行了一次单独的读写分离的示例配置，但语句还是和上面的一样，从语句上没有看出明显的读从库

感觉可能有下面的几种情况：

- 1.已经走的读库，只是打印的语句设置如此
- 2.数据库需要额外的配置

这个后面再研究下.....

### 数据加密
我们就简单使用这个示例工程进行尝试：examples/shardingsphere-jdbc-example/governance-example/governance-spring-boot-mybatis-example

#### 1.数据库建立
```sql
CREATE SCHEMA IF NOT EXISTS demo_ds;
```

#### 2.配置修改
修改配置：examples/shardingsphere-jdbc-example/governance-example/governance-spring-boot-mybatis-example/src/main/resources/application.properties

```yaml
spring.profiles.active=local-zookeeper-encryp
```

修改配置：examples/shardingsphere-jdbc-example/governance-example/governance-spring-boot-mybatis-example/src/main/resources/application-local-zookeeper-encrypt.properties

修改数据库密码即可

#### 3.运行
示例运行过后都会清理数据，先在代码中把相关的数据库清理给关掉：

examples/example-core/example-api/src/main/java/org/apache/shardingsphere/example/core/api/ExampleExecuteTemplate.java

```java
public final class ExampleExecuteTemplate {
    
    public static void run(final ExampleService exampleService) throws SQLException {
        try {
            exampleService.initEnvironment();
            exampleService.processSuccess();
        } finally {
	    // 注释掉
//            exampleService.cleanEnvironment();
        }
    }
    
    public static void runFailure(final ExampleService exampleService) throws SQLException {
        try {
            exampleService.initEnvironment();
            exampleService.processFailure();
        } finally {
            exampleService.cleanEnvironment();
        }
    }
}
```

examples/example-core/example-spring-mybatis/src/main/java/org/apache/shardingsphere/example/core/mybatis/service/OrderServiceImpl.java

```java
@Service
@Primary
public class OrderServiceImpl implements ExampleService {
    ......
    
    @Override
    @Transactional
    public void processSuccess() throws SQLException {
        System.out.println("-------------- Process Success Begin ---------------");
        List<Long> orderIds = insertData();
        printData();
	// 注释掉
//        deleteData(orderIds);
        printData();
        System.out.println("-------------- Process Success Finish --------------");
    }

    ......
}
```

跑来了，就看到了大量的日志，由上面提到的SQL之类的

上面我们关闭了order相关的数据清理代码，我们看看数据库里面的字段张啥样的，大致如下，最后一个字段看的出来是加密了的


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/769f914ed46e4a82b3f421d124582640~tplv-k3u1fbpfcp-watermark.image)

## 总结
本篇文件运行了源码示例中的数据分片（分库分表）、读写分离、数据加密相关的

其中数据库分片和数据加密符合我们的初步预期，但读写分离有点怪，有点不太预期，这个后面要研究下