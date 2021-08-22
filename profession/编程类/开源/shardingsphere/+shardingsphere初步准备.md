# ShardingSphere源码解析 初步准备
***
## 简介
源码阅读解析前，肯定是要对其有一个初步的了解，其用于解决问题，用于哪些场景。并上手本地跑一跑官方示例之类，开始阅读解析的第一步，为后面做准备。

## 阅读解析准备
GitHub和项目官网是了解的好途径：

- [Github address](https://github.com/apache/shardingsphere)
- [官网](https://shardingsphere.apache.org/document/current/cn/overview/)

初步看GitHub的介绍，Apache ShardingSphere是一个由一组分布式数据库解决方案组成的开源生态系统，下面是介绍：

> Apache ShardingSphere is an open-source ecosystem consisting of a set of distributed database solutions, including 3 independent products, JDBC, Proxy & Sidecar (Planning). They all provide functions of data scale-out, distributed transaction and distributed governance, applicable in a variety of situations such as Java isomorphism, heterogeneous language and cloud-native.
> 来源GitHub官网：https://github.com/apache/shardingsphere

更多相关的介绍就自行查看了，刚开始文档肯定是要自己扫一遍，虽然看不太懂，但起码心中有些印象，知道项目大体是做啥的，有哪些模块之类的

下面是本地环境准备阶段：

- 克隆代码到本地
- 运行体验代码示例

### 克隆代码到本地
一般来说直接 git clone 就基本解决了，但国内的特殊环境，ShardingSphere又太大了，没有点其他东西，还拉不下来

注：ShardingSphere有些文件名太长了，需要运行下面的命令进行设置，不然拉不下来：

```shell script
# git clone 文件名太长
git config --global core.longpaths true
```

目前解决克隆速度慢有下面几种方法：

#### 1.只克隆最新的一层：使用--depth=1，只拉取代码最新的一层，这样不会拉取代码历史，之前尝试过此方法可行，多试几次基本都能拉下来

```shell script
git clone --depth=1 https://github.com/apache/shardingsphere
```

#### 2.使用Chrome Github 下载插件，使用其提供的地址进行克隆，很快很快，也是博主一直以来使用的方法
此方法嗖嗖的，快滴很，如果在线访问安装不了的话，使用博主提供的网盘链接或者去其他地方下载后，本地安装也可以，本地安装的教程链接也放到下面了，可能还需要自己进行一些处理

- [Github 地址](https://github.com/fhefh2015/Fast-GitHub)
- [Github 加速插件地址](https://chrome.google.com/webstore/detail/github%E5%8A%A0%E9%80%9F/mfnkflidjnladnkldfonnaicljppahpg?hl=zh-CN&authuser=1)
- 百度网盘链接: https://pan.baidu.com/s/1Fcm2immxryeDJfgilxITug 提取码: 7pxk
- [国内离线安装 Chrome 扩展程序的方法总结](https://zhuanlan.zhihu.com/p/80305764)
- [Chrome 手动安装.crx插件](https://www.jianshu.com/p/bb51dc91b93a)

安装完成后，进入GitHub网页，获取加速地址，克隆即可

```shell script
git clone https://github.com.cnpmjs.org/apache/shardingsphere.git
```

注：如果是fork的项目，后面需要提pr的，需要将地址修改回非加速地址

#### 3.从Gitee上clone
简单学习之类的，从别人同步到Gitee的项目克隆，因为是国内，也是嗖嗖的

#### 4.下载源码包
也可以直接在GitHub上下载源码包，虽然本地解压后好像没有git历史了，但也够学习之用了，当然，速度也是慢吞吞的......


### 示例代码运行
使用IDEA打开后，先本地使用Maven工具，clean、install操作下，注意跳过测试

这个如果不使用国内镜像的话，也是比较耗时的，趁着这段时间下楼跑个步，冲个澡后回来继续

在根目录下有个example目录，右键选择作为Maven工程加载，然后再clean、install一下

#### 数据库准备
这里就使用docker启动一个吧，方便，用户和密码都是root

```shell script
docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root -d mysql:latest
```

#### 数据库初始化
使用官方提供的数据库初始化脚本：examples/src/resources/manual_schema.sql

表之类的，ShardingSphere好像自己在代码里面操作，后面探索下这个

#### 修改MySQL驱动依赖
提供的示例稍有点问题，在MySQL版本是8+时，会报下面的错误：

> Unable to load authentication plugin ‘caching_sha2_password‘

修改MySQL的驱动版本，Maven配置文件是：examples/pom.xml

修改内容如下：

```yaml
<mysql-connector-java.version>8.0.22</mysql-connector-java.version>
```

#### 修改配置
修改配置文件，这里修改密码即可，大致如下：examples/shardingsphere-jdbc-example/sharding-example/sharding-raw-jdbc-example/src/main/resources/META-INF/sharding-databases-range.yaml

```yaml
dataSources:
  ds_0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3306/demo_ds_0?serverTimezone=UTC&useSSL=false&useUnicode=true&characterEncoding=UTF-8
    username: root
    password: root
  ds_1:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3306/demo_ds_1?serverTimezone=UTC&useSSL=false&useUnicode=true&characterEncoding=UTF-8
    username: root
    password: roo
```

#### 运行示例
这里选择启动类：examples/shardingsphere-jdbc-example/sharding-example/sharding-raw-jdbc-example/src/main/java/org/apache/shardingsphere/example/sharding/raw/jdbc/YamlRangeConfigurationExampleMain.java

通过看代码能准确知道这个启动类的相关配置文件，就直接选它了

运行起来输出大致如下：

```
[INFO ] 2021-08-22 07:38:28,331 --main-- [com.zaxxer.hikari.HikariDataSource] HikariPool-1 - Starting... 
[WARN ] 2021-08-22 07:38:28,370 --main-- [com.zaxxer.hikari.util.DriverDataSource] Registered driver with driverClassName=com.mysql.jdbc.Driver was not found, trying direct instantiation. 
[INFO ] 2021-08-22 07:38:28,871 --main-- [com.zaxxer.hikari.HikariDataSource] HikariPool-1 - Start completed. 
[INFO ] 2021-08-22 07:38:28,893 --main-- [com.zaxxer.hikari.HikariDataSource] HikariPool-2 - Starting... 
[WARN ] 2021-08-22 07:38:28,893 --main-- [com.zaxxer.hikari.util.DriverDataSource] Registered driver with driverClassName=com.mysql.jdbc.Driver was not found, trying direct instantiation. 
[INFO ] 2021-08-22 07:38:28,908 --main-- [com.zaxxer.hikari.HikariDataSource] HikariPool-2 - Start completed. 
-------------- Process Success Begin ---------------
---------------------------- Insert Data ----------------------------
---------------------------- Print Order Data -----------------------
order_id: 636106123225051136, user_id: 1, address_id: 1, status: INSERT_TEST
order_id: 636106123476709376, user_id: 2, address_id: 2, status: INSERT_TEST
order_id: 636106123636092928, user_id: 3, address_id: 3, status: INSERT_TEST
order_id: 636106123833225216, user_id: 4, address_id: 4, status: INSERT_TEST
order_id: 636106124013580288, user_id: 5, address_id: 5, status: INSERT_TEST
order_id: 636106124164575232, user_id: 6, address_id: 6, status: INSERT_TEST
order_id: 636106124336541696, user_id: 7, address_id: 7, status: INSERT_TEST
order_id: 636106124508508160, user_id: 8, address_id: 8, status: INSERT_TEST
order_id: 636106124672086016, user_id: 9, address_id: 9, status: INSERT_TEST
order_id: 636106124839858176, user_id: 10, address_id: 10, status: INSERT_TEST
---------------------------- Print OrderItem Data -------------------
order_item_id:636106123409600513, order_id: 636106123225051136, user_id: 1, status: INSERT_TEST
order_item_id:636106123543818241, order_id: 636106123476709376, user_id: 2, status: INSERT_TEST
order_item_id:636106123749339137, order_id: 636106123636092928, user_id: 3, status: INSERT_TEST
order_item_id:636106123929694209, order_id: 636106123833225216, user_id: 4, status: INSERT_TEST
order_item_id:636106124093272065, order_id: 636106124013580288, user_id: 5, status: INSERT_TEST
order_item_id:636106124248461313, order_id: 636106124164575232, user_id: 6, status: INSERT_TEST
order_item_id:636106124433010689, order_id: 636106124336541696, user_id: 7, status: INSERT_TEST
order_item_id:636106124584005633, order_id: 636106124508508160, user_id: 8, status: INSERT_TEST
order_item_id:636106124747583489, order_id: 636106124672086016, user_id: 9, status: INSERT_TEST
order_item_id:636106124919549953, order_id: 636106124839858176, user_id: 10, status: INSERT_TEST
---------------------------- Delete Data ----------------------------
---------------------------- Print Order Data -----------------------
---------------------------- Print OrderItem Data -------------------
-------------- Process Success Finish --------------
Disconnected from the target VM, address: '127.0.0.1:53730', transport: 'socket'

Process finished with exit code 
```

## 总结
本篇做了下源码阅读前的准备，大致扫了下官方文档，虽然还是比较懵，但起码有个大体印象

然后拉取官方的代码，在本地成功运行了，为后面的分析和调试打下基础

## 参考链接
- [ShardingSphere](https://shardingsphere.apache.org/document/current/cn/overview/)
- [ShardingSphere-JDBC小伙伴之Mybatis-plus](https://blog.csdn.net/weixin_37669199/article/details/111180241)
- [6.4.1.2 Caching SHA-2 Pluggable Authentication](https://dev.mysql.com/doc/refman/8.0/en/caching-sha2-pluggable-authentication.html)
- [Chrome导出扩展程序（插件）](https://www.jianshu.com/p/ad03952344e7)