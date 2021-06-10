# ApacheRocketMQ源码解析（一）概览
***
## 简介
&ensp;&ensp;&ensp;&ensp;由于写消息队列的Demo卡壳了，就想看看别人消息队列数组和锁的使用那块是如何使用的，看能不能借鉴一下

&ensp;&ensp;&ensp;&ensp;第一次接触就按照学习流程，将官方文档通读一遍

## 概览
&ensp;&ensp;&ensp;&ensp;下面是官网地址，中文的，基本概念倒是看懂了，但设计和架构啥的懂的皮毛，深层意思看不懂，稍微读一下就好了，后面有疑问回过头来看

- [RocketMQ中文官方文档](https://github.com/apache/rocketmq/tree/master/docs/cn)

&ensp;&ensp;&ensp;&ensp;概念和特性，需要稍微读认真一点。后面的有个大概了解一下就好，有个印象。毕竟写MQ Demo目前只先写单机的