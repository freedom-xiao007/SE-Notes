# ShardingSphere Proxy 初步体验
***
## 简介
在上篇文章中，体验了ShardingSphere JDBC的数据分片、读写分离、数据加密，本篇文章就来探索下ShardingSphere Proxy相关的功能

## 示例运行
ShardingSphere Proxy相对来说还是比较陌上的，首先肯定是官方文档了解一波：

- [ShardingSphere概览](https://shardingsphere.apache.org/document/current/cn/overview/)

数据分片、读写分离、数据加密的说明和配置和ShardingSphere JDBC一样，参考上篇文章即可

ShardingSphere JDBC如果要接入的话，从上篇文章中可以看出，其是需要改变现有业务代码的，是侵入式的

ShardingSphere Proxy感觉更像一个代码，只需要改变数据库的连接配置，是非侵入式的。但也增加了整个系统的复杂度，各有利弊吧

### ShardingSphere Proxy启动相关
首先找到启动的地方：shardingsphere-proxy/shardingsphere-proxy-bootstrap/src/main/java/org/apache/shardingsphere/proxy/Bootstrap.java

上面就是启动类，但还不能进行启动，需要先将server.yaml进行开启：shardingsphere-proxy/shardingsphere-proxy-bootstrap/src/main/resources/conf/server.yaml

下面的注释全部放开：

其中配置了ShardingSphere连接是用户名和密码，下面的配置文件中就配置了两个：root和sharding

sql-show改为true，看日志利于排查问题（一个示例跑起来问题真不少啊）

```yaml
rules:
  - !AUTHORITY
    users:
      - root@%:root
      - sharding@:sharding
    provider:
      type: NATIVE

props:
  max-connections-size-per-query: 1
  executor-size: 16  # Infinite by default.
  proxy-frontend-flush-threshold: 128  # The default value is 128.
    # LOCAL: Proxy will run with LOCAL transaction.
    # XA: Proxy will run with XA transaction.
    # BASE: Proxy will run with B.A.S.E transaction.
  proxy-transaction-type: LOCAL
  xa-transaction-manager-type: Atomikos
  proxy-opentracing-enabled: false
  proxy-hint-enabled: false
  sql-show: true
  check-table-metadata-enabled: false
  lock-wait-timeout-milliseconds: 50000 # The maximum time to wait for a lock
  show-process-list-enabled: false
    # Proxy backend query fetch size. A larger value may increase the memory usage of ShardingSphere Proxy.
    # The default value is -1, which means set the minimum value for different JDBC drivers.
  proxy-backend-query-fetch-size: -1
  check-duplicate-table-enabled: fals
```

相关的配置文件位于：shardingsphere-proxy/shardingsphere-proxy-bootstrap/src/main/resources/conf

下面有各个示例开启的配置文件，先不进行开启，跟着下面的步骤进行操作即可

使用docker启动一个数据库：

```shell script
docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root -d mysql:latest
```

注：本地采用单个测试，所以在测试一个特性的时候，其他特性的配置文件是注释掉的

注：连接ShardingSphere Proxy时，数据库名称是Proxy的逻辑名称：schemaName，注意进行相应的修改

### 数据分片
#### 1.数据库初始化
用下面的语句建立相关的数据库

```sql
CREATE SCHEMA IF NOT EXISTS demo_ds_0;
CREATE SCHEMA IF NOT EXISTS demo_ds_1;
```

#### 2.ShardingSphere Proxy配置
放开配置：shardingsphere-proxy/shardingsphere-proxy-bootstrap/src/main/resources/conf/config-sharding.yaml

就是简单修改密码，大致配置如下：

```yaml
schemaName: sharding_db

dataSources:
  ds_0:
    url: jdbc:mysql://127.0.0.1:3306/demo_ds_0?serverTimezone=UTC&useSSL=false
    username: root
    password: root
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1
  ds_1:
    url: jdbc:mysql://127.0.0.1:3306/demo_ds_1?serverTimezone=UTC&useSSL=false
    username: root
    password: root
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1

rules:
- !SHARDING
  tables:
    t_order:
      actualDataNodes: ds_${0..1}.t_order_${0..1}
      tableStrategy:
        standard:
          shardingColumn: order_id
          shardingAlgorithmName: t_order_inline
      keyGenerateStrategy:
        column: order_id
        keyGeneratorName: snowflake
    t_order_item:
      actualDataNodes: ds_${0..1}.t_order_item_${0..1}
      tableStrategy:
        standard:
          shardingColumn: order_id
          shardingAlgorithmName: t_order_item_inline
      keyGenerateStrategy:
        column: order_item_id
        keyGeneratorName: snowflake
  bindingTables:
    - t_order,t_order_item
  defaultDatabaseStrategy:
    standard:
      shardingColumn: user_id
      shardingAlgorithmName: database_inline
  defaultTableStrategy:
    none:

  shardingAlgorithms:
    database_inline:
      type: INLINE
      props:
        algorithm-expression: ds_${user_id % 2}
    t_order_inline:
      type: INLINE
      props:
        algorithm-expression: t_order_${order_id % 2}
    t_order_item_inline:
      type: INLINE
      props:
        algorithm-expression: t_order_item_${order_id % 2}

  keyGenerators:
    snowflake:
      type: SNOWFLAKE
      props:
        worker-id: 123
```

配置完成后，重启

#### 3.示例运行
修改配置：examples/shardingsphere-proxy-example/shardingsphere-proxy-boot-mybatis-example/src/main/resources/application.properties

让访问的数据库名和密码对上，配置大致如下：

```yaml
mybatis.config-location=classpath:META-INF/mybatis-config.xml

spring.datasource.type=com.zaxxer.hikari.HikariDataSource
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3307/sharding_db?useServerPrepStmts=true&cachePrepStmts=true
spring.datasource.username=root
spring.datasource.password=root
```

启动后，我们就看到相关的ShardingSphere Proxy的日志：

```
Logic SQL: select @@session.transaction_read_only
SQLStatement: MySQLSelectStatement(limit=Optional.empty, lock=Optional.empty, window=Optional.empty)
Actual SQL: ds_0 ::: select @@session.transaction_read_only
Logic SQL: INSERT INTO t_order_item (order_id, user_id, status) VALUES (?, ?, ?);
SQLStatement: MySQLInsertStatement(setAssignment=Optional.empty, onDuplicateKeyColumns=Optional.empty)
Actual SQL: ds_0 ::: INSERT INTO t_order_item_0 (order_id, user_id, status, order_item_id) VALUES (?, ?, ?, ?); ::: [637039855574429696, 10, INSERT_TEST, 637039855616372737]
Logic SQL: select @@session.transaction_read_only
SQLStatement: MySQLSelectStatement(limit=Optional.empty, lock=Optional.empty, window=Optional.empty)
Actual SQL: ds_0 ::: select @@session.transaction_read_only
Logic SQL: SELECT * FROM t_order;
SQLStatement: MySQLSelectStatement(limit=Optional.empty, lock=Optional.empty, window=Optional.empty)
Actual SQL: ds_0 ::: SELECT * FROM t_order_0 ORDER BY order_id ASC ;
Actual SQL: ds_0 ::: SELECT * FROM t_order_1 ORDER BY order_id ASC ;
Actual SQL: ds_1 ::: SELECT * FROM t_order_0 ORDER BY order_id ASC ;
Actual SQL: ds_1 ::: SELECT * FROM t_order_1 ORDER BY order_id ASC ;
```

从上面能大致看的处理，分片策略是起效的

接着运行下面的示例，记得环境ShardingSphere Proxy的配置

### 读写分离
#### 1.数据库初始化
用下面的语句建立相关的数据库

```sql
CREATE SCHEMA IF NOT EXISTS demo_write_ds;
CREATE SCHEMA IF NOT EXISTS demo_read_ds_0;
CREATE SCHEMA IF NOT EXISTS demo_read_ds_1;

CREATE TABLE IF NOT EXISTS demo_read_ds_0.t_order (order_id BIGINT NOT NULL AUTO_INCREMENT, user_id INT NOT NULL, status VARCHAR(50), PRIMARY KEY (order_id));
CREATE TABLE IF NOT EXISTS demo_read_ds_1.t_order (order_id BIGINT NOT NULL AUTO_INCREMENT, user_id INT NOT NULL, status VARCHAR(50), PRIMARY KEY (order_id));
CREATE TABLE IF NOT EXISTS demo_read_ds_0.t_order_item (order_item_id BIGINT NOT NULL AUTO_INCREMENT, order_id BIGINT NOT NULL, user_id INT NOT NULL, status VARCHAR(50), PRIMARY KEY (order_item_id));
CREATE TABLE IF NOT EXISTS demo_read_ds_1.t_order_item (order_item_id BIGINT NOT NULL AUTO_INCREMENT, order_id BIGINT NOT NULL, user_id INT NOT NULL, status VARCHAR(50), PRIMARY KEY (order_item_id));
```

#### 2.ShardingSphere Proxy配置
我们放开配置：shardingsphere-proxy/shardingsphere-proxy-bootstrap/src/main/resources/conf/config-readwrite-splitting.yaml

放开其配置即可,然后改改密码，大致如下：

记住我们的数据库是： readwrite_splitting_db

```yaml
schemaName: readwrite_splitting_db

dataSources:
  write_ds:
    url: jdbc:mysql://127.0.0.1:3306/demo_write_ds?serverTimezone=UTC&useSSL=false
    username: root
    password: root
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1
  read_ds_0:
    url: jdbc:mysql://127.0.0.1:3306/demo_read_ds_0?serverTimezone=UTC&useSSL=false
    username: root
    password: root
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1
  read_ds_1:
    url: jdbc:mysql://127.0.0.1:3306/demo_read_ds_1?serverTimezone=UTC&useSSL=false
    username: root
    password: root
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1

rules:
- !READWRITE_SPLITTING
  dataSources:
    pr_ds:
      writeDataSourceName: write_ds
      readDataSourceNames:
        - read_ds_0
        - read_ds_1
```

配置完成后，重启

#### 3.示例运行
使用官方示例：examples/shardingsphere-proxy-example/shardingsphere-proxy-boot-mybatis-example/src/main/java/org/apache/shardingsphere/example/proxy/spring/boot/mybatis/ProxySpringBootStarterExample.java

要修改下配置：examples/shardingsphere-proxy-example/shardingsphere-proxy-boot-mybatis-example/src/main/resources/application.properties

将数据库名改成 readwrite_splitting_db，大致如下：

```yaml
mybatis.config-location=classpath:META-INF/mybatis-config.xml

spring.datasource.type=com.zaxxer.hikari.HikariDataSource
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3307/readwrite_splitting_db?useServerPrepStmts=true&cachePrepStmts=true
spring.datasource.username=root
spring.datasource.password=root
```

然后启动，我们就看到日志，日志是ShardingSphere Proxy的：

```
SQLStatement: MySQLInsertStatement(setAssignment=Optional.empty, onDuplicateKeyColumns=Optional.empty)
Actual SQL: write_ds ::: INSERT INTO t_order_item (order_id, user_id, status) VALUES (?, ?, ?); ::: [20, 10, INSERT_TEST]
Logic SQL: select @@session.transaction_read_only
SQLStatement: MySQLSelectStatement(limit=Optional.empty, lock=Optional.empty, window=Optional.empty)
Actual SQL: read_ds_0 ::: select @@session.transaction_read_only
Logic SQL: SELECT * FROM t_order;
SQLStatement: MySQLSelectStatement(limit=Optional.empty, lock=Optional.empty, window=Optional.empty)
Actual SQL: read_ds_1 ::: SELECT * FROM t_order
```

从上面的日志可以看出，写入是成功了的，查询的时候也走了从库，说明我们的读写分离是有效的

但也证明我们昨天的读写分离失败了，这个后面得研究下

在启动也报了个错：Column index out of range.

感觉是示例问题，暂时不管（其实尝试搞了下，没搞定，哎）

这不测试完了，尝试下一个的时候，记得将配置还原

### 数据加密
#### 1.数据库初始化
执行下面的语句对数据库进行初始化：

```sql
CREATE SCHEMA IF NOT EXISTS demo_ds_0;
CREATE SCHEMA IF NOT EXISTS demo_ds_1;
```

#### 2.ShardingSphere Proxy配置
放开加密配置：shardingsphere-proxy/shardingsphere-proxy-bootstrap/src/main/resources/conf/config-encrypt.yaml

声明了数据库名称：encrypt_db

对下面两个字段进行加密：user_id/order_id

配置大致如下：

```yaml
schemaName: encrypt_db

dataSources:
  ds_0:
    url: jdbc:mysql://127.0.0.1:3306/demo_ds_0?serverTimezone=UTC&useSSL=false
    username: root
    password: root
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1
  ds_1:
    url: jdbc:mysql://127.0.0.1:3306/demo_ds_1?serverTimezone=UTC&useSSL=false
    username: root
    password: root
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1

rules:
- !ENCRYPT
  encryptors:
    aes_encryptor:
      type: AES
      props:
        aes-key-value: 123456abc
    md5_encryptor:
      type: MD5
  tables:
    t_encrypt:
      columns:
        user_id:
          plainColumn: user_plain
          cipherColumn: user_cipher
          encryptorName: aes_encryptor
        order_id:
          cipherColumn: order_cipher
          encryptorName: md5_encrypto
```

配置修改后，进行重启（记得其他配置文件的配置要注释掉）

#### 3.示例运行
修改配置：examples/shardingsphere-proxy-example/shardingsphere-proxy-boot-mybatis-example/src/main/resources/application.properties

修改数据库密码和数据库名为：encrypt_db

大致配置为：

```yaml
mybatis.config-location=classpath:META-INF/mybatis-config.xml

spring.datasource.type=com.zaxxer.hikari.HikariDataSource
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3307/encrypt_db?useServerPrepStmts=true&cachePrepStmts=true
spring.datasource.username=root
spring.datasource.password=root
```

运行启动，从日志看到我们确实跑起来，日志看起来也是正常

## 总结
本篇文件使用ShardingSphere Proxy，单独运行尝试了数据分片、数据加密、读写分离

在示例中，好像三者是分离的，不同的配置文件，不同的逻辑数据库名称

那三者能结合或者两两组合吗，从文档上来看，应该是可以，后面探索下