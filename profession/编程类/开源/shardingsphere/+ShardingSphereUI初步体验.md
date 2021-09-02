# ShardingSphere UI 初步体验
***
## 简介
在上两篇文章中，尝试了ShardingSphere JDBC和Proxy的相关功能，本篇进行探索ShardingSphere的UI组件部分

## 示例运行
这个应该是一个管理配置之类的东西，国际惯例，先瞅一瞅文档：

- [ShardingSphere-UI](https://shardingsphere.apache.org/document/current/cn/user-manual/shardingsphere-ui/)

上面的官方文档中对其做了一个大致的介绍，下面我尝试运行下：

### 1.代码拉取
运行下面的命令进行相关代码的拉取：

```shell script
git clone https://github.com.cnpmjs.org/apache/shardingsphere-ui.git
```

### 2.编译
直接运行命令，好像一直报错，那只能手动去安装了

下面Nodejs：https://nodejs.org/en/download/ ，并进行相应的安装

配置一下源(安装完成后，重新打开命令行终端，命令是生效的），然后进行相关依赖的安装：

```shell script
npm config set registry https://registry.npm.taobao.org
npm install
npm run dev
# 运行过程中可能会报错，运行下面的命令，再次运行即可
npm rebuild node-sass
npm run dev
```

### 3.后端代码
使用IDEA打开，配置好Java8和Maven

实现修改下跟目录下的配置：pom.xml

修改下ShardingSphere相关依赖版本为：<shardingsphere.version>5.0.0-alpha</shardingsphere.version>

然后运行后台代码：shardingsphere-ui-backend/src/main/java/org/apache/shardingsphere/ui/Bootstrap.java

成功跑了起来

### 4.页面登录访问
在第二步中前端已经跑起来了，访问显示的地址：http://localhost:8080/

进去默认就填好了用户名和密码，点击登录即可，登录后如下图：

### 5.Proxy连接
#### 启动Zookeeper
使用Docker启动一个zk：

```shell script
docker run -dit --name zk -p 2181:2181 zookeeper
```

#### 将源码版本切换到5.0.0-alpha
5.0.0-beta的Proxy注册会有问题，数据匹配不上，我们这里使用5.0.0-alpha版本的，使用命令行，将代码重置到相关版本即可，命令如下：

```shell script
git checkout 5dc690c2227571e83beada277dbb2dfb43c29427
```

打了相关标签的，成功的话，会显示下面的一行：

> HEAD is now at 5dc690c222 [maven-release-plugin] prepare release 5.0.0-alpha

#### 修改配置，启动Proxy
接下来我们就开始修改配置启动Proxy

首先修改server.yaml：shardingsphere-proxy\shardingsphere-proxy-bootstrap\src\main\resources\conf\server.yaml

放开下面的配置，记住我们的名字：governance_ds

```yaml
governance:
  name: governance_ds
  registryCenter:
    type: ZooKeeper
    serverLists: localhost:2181
    props:
      retryIntervalMilliseconds: 5000
      timeToLiveSeconds: 60
      maxRetries: 3
      operationTimeoutMilliseconds: 5000
  overwrite: false

authentication:
  users:
    root:
      password: root
    sharding:
      password: sharding
      authorizedSchemas: sharding_db

props:
  max-connections-size-per-query: 1
  acceptor-size: 16  # The default value is available processors count * 2.
  executor-size: 16  # Infinite by default.
  proxy-frontend-flush-threshold: 128  # The default value is 128.
    # LOCAL: Proxy will run with LOCAL transaction.
    # XA: Proxy will run with XA transaction.
    # BASE: Proxy will run with B.A.S.E transaction.
  proxy-transaction-type: LOCAL
  proxy-opentracing-enabled: false
  proxy-hint-enabled: false
  query-with-cipher-column: true
  sql-show: true
  check-table-metadata-enabled: false
```

再讲数据加密配置放开：shardingsphere-proxy\shardingsphere-proxy-bootstrap\src\main\resources\conf\config-encrypt.yaml

```yaml
schemaName: encrypt_db

dataSource:
  url: jdbc:mysql://127.0.0.1:3306/demo_ds?serverTimezone=UTC&useSSL=false
  username: root
  password: root
  connectionTimeoutMilliseconds: 30000
  idleTimeoutMilliseconds: 60000
  maxLifetimeMilliseconds: 1800000
  maxPoolSize: 50

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
          encryptorName: md5_encryptor
```

在启动的时候，还是遇到了问题，明明没有放开数据分片的配置，但Proxy还是默认加载了数据分片的配置，对数据分片配置进行修改了也不行，由于默认是无密码连接的，导致一直连接不上

于是只能跟踪代码，将默认的数据库连接密码都改为root，让Proxy暂时能跑起来：

修改的文件是：shardingsphere-proxy\shardingsphere-proxy-backend\src\main\java\org\apache\shardingsphere\proxy\backend\communication\jdbc\datasource\factory\JDBCRawBackendDataSourceFactory.java

将密码暂时都设置为root：

```java
public final class JDBCRawBackendDataSourceFactory implements JDBCBackendDataSourceFactory {
    
    @SuppressWarnings({"unchecked", "rawtypes"})
    @Override
    public DataSource build(final String dataSourceName, final DataSourceParameter dataSourceParameter) {
        HikariConfig config = new HikariConfig();
        String driverClassName = JDBCDriverURLRecognizerEngine.getJDBCDriverURLRecognizer(dataSourceParameter.getUrl()).getDriverClassName();
        validateDriverClassName(driverClassName);
        config.setDriverClassName(driverClassName);
        config.setJdbcUrl(dataSourceParameter.getUrl());
        config.setUsername(dataSourceParameter.getUsername());
//        config.setPassword(dataSourceParameter.getPassword());
        config.setPassword("root");
        config.setConnectionTimeout(dataSourceParameter.getConnectionTimeoutMilliseconds());
        config.setIdleTimeout(dataSourceParameter.getIdleTimeoutMilliseconds());
        config.setMaxLifetime(dataSourceParameter.getMaxLifetimeMilliseconds());
        config.setMaximumPoolSize(dataSourceParameter.getMaxPoolSize());
        config.setMinimumIdle(dataSourceParameter.getMinPoolSize());
        config.setReadOnly(dataSourceParameter.isReadOnly());
        DataSource result = new HikariDataSource(config);
        Optional<JDBCParameterDecorator> decorator = findJDBCParameterDecorator(result);
        return decorator.isPresent() ? decorator.get().decorate(result) : result;
    }
}
```

但是为啥默认一直有数据分片的配置，时间不够，后面再跟踪研究下

这样，我们就可以启动Proxy了，在ZK中查看，我们生成了节点：governance_ds


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8642f0c0ef984c28a131153e8129056d~tplv-k3u1fbpfcp-watermark.image)

#### 注册中心设置
在页面上添加一个注册中心，操作大致如下，填入Proxy中我们配置的名称：governance_ds，注意一定要对应上，这个涉及到ZK的读取路径

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d36cbd73645046a0846a0d38a7a76818~tplv-k3u1fbpfcp-watermark.image)

添加完成后，点击激活，后面就会显示已经连接了注册中心，如下图：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/20121ff1076f4471b528d3ae9427338b~tplv-k3u1fbpfcp-watermark.image)

在配置管理中，我们能看到相关的配置：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/78ccbce112e94f849ea3d471aad3c121~tplv-k3u1fbpfcp-watermark.image)

运行状态中也看到了我们的Proxy节点：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e281e451e7140eb8101e82793d924ad~tplv-k3u1fbpfcp-watermark.image)

## 总结
本篇中体验了下ShardingSphere UI的基本功能，使用Proxy连接了上去

后面的动态数据同步，由于时间问题，还没来得及去尝试，但问题不大，感兴趣的老哥可以自定尝试

核心的组件基本上体验完了，并且都是源码上直接启动的，后面准备开始源码上的阅读了