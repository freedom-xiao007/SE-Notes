# Mybatis3 源码解析系列
***

这是我参与2022首次更文挑战的第18天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7052884569032392740)

## 简介
Mybatis作为一个优秀的Java持久化框架，在我们的日常工作中相信都会用到，本次源码解析系列，就开始探索下Mybatis

## 总结
在MyBatis的学习中，首先通读了《MyBatis3源码深度解析》一遍，然后抱着如何去写一个基本功能的MyBatis框架的想法，又读了2-3遍

心中有了大致的想法，然后再去通过MyBatis的示例去走一遍源码，注重关注了一些在写Demo中可能会遇到的细节点

后面花了两三天的时间，把基本功能的框架Dome给写了出来，各个感觉还是可以的，达到了自己预期的目标

下面再总结下MyBatis的学习：

下面一个图，来源于：《MyBatis3源码深度解析》基本涵盖了MyBatis的核心：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e4f056ad21a74e7eafa906e63f4a9c43~tplv-k3u1fbpfcp-watermark.image?)

最右侧的是全局配置 Configuration：这里负责前期Mapper的解析和TypeHandler注册相关的，在初始化阶段，把在后期SQL查询前的参数解析和结果转换时需要用到的东西先存下来，便于后面获取用于处理

左侧是MyBatis的核心类：

- SQLSession：可以算是整个Mybatis的入口，数据库源与和Mapper的代理对象从这里进行获取
- Executor：语句执行入口
- StatementHandler：可以算是JDBC中对于Statement的封装，主要是语句生成相关方面的处理
- ParameterHandler：SQL查询时参数转换处理,如果有参数则调用TypeHandler相关逻辑
- ResultSetHandler：负责SQL结果的处理，如果有返回结果需要处理，则调用TypeHandler相关逻辑
- TypeHandler：负责JavaType与jdbcType的相关转换

感觉核心逻辑主线就是这些了，自己在Demo中除了ParameterHandler没有进行实现，其他基本都有体现

当然，读代码时候发现，细节还是挺多的，还有很多的地方没有仔细去研究，目前就简单看了下，有个印象，方便如果以后遇到问题，也能去定位后，结合问题场景仔细研究

在研究的过程中发现这些数据库的相关的框架，基本都是基于JDBC规范的Statement等去做文章的，比如MyBatis可以结合HikariCP，再结合Shardingsphere，感觉挺有意思，自己之前写过一篇基于这三者做多数据源的文章：[ShardingSphere JDBC 分库实现多数据库源](https://juejin.cn/post/7056118707759775781)。写完还有点懵，现在就知道哪些Bean定义的相关原理和作用，做到了心中有数

本系列有源码解析部分和Demo实现部分，涉及到的范围就是上面的核心逻辑主线部分，还有如动态SQL（这个在日常开发中经常使用）之类没有去探索，但大致原理看书了解一些，留待以后有空再研究

## 解析文章目录
- [MyBatis3源码解析（1）探索准备](https://juejin.cn/post/7058354949209456653)
- [MyBatis3源码解析(2)数据库连接](https://juejin.cn/post/7061031527001358349)
- [MyBatis3源码解析(3)查询语句执行](https://juejin.cn/post/7061427063793647647/)
- [MyBatis3源码解析(4)参数解析](https://juejin.cn/post/7061763240501444615)
- [MyBatis3源码解析(5)查询结果处理](https://juejin.cn/post/7062333998348894244/)
- [MyBatis3源码解析(6)TypeHandler使用](https://juejin.cn/post/7062858058535272478/)
- [MyBatis3源码解析(7)TypeHandler注册与获取](https://juejin.cn/post/7063234640848519175/)
- [MyBatis3源码解析(8)MyBatis与Spring的结合](https://juejin.cn/post/7063649335686201381/)

## Demo 编写
完整的工程已放到GitHub上：https://github.com/lw1243925457/MybatisDemo/tree/master/

- [MyBatis Demo 编写（1）基础功能搭建](https://juejin.cn/post/7064351012022124580/)
- [MyBatis Demo 编写（2）结果映射转换处理](https://juejin.cn/post/7064907905669005342/)

## 参考链接
- 《MyBatis3源码深度解析》：这本书确实不错，通读一两遍后，自己探索Debug，有很多的帮助