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

修改下ShardingSphere相关依赖版本为：<shardingsphere.version>5.0.0-beta</shardingsphere.version>

然后运行后台代码：shardingsphere-ui-backend/src/main/java/org/apache/shardingsphere/ui/Bootstrap.java

成功跑了起来

### 4.页面登录访问
在第二步中前端已经跑起来了，访问显示的地址：http://localhost:8080/

进去默认就填好了用户名和密码，点击登录即可，登录后如下图：


使用Docker启动一个zk：

```shell script
docker run -dit --name zk -p 2181:2181 zookeeper
```

在页面上添加一个注册中心，操作大致如下：


添加完成后，点击激活，后面就会显示已经连接了注册中心，如下图：

```shell script
git checkout 5dc690c2227571e83beada277dbb2dfb43c29427
```