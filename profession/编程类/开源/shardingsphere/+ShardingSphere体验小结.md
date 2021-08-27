# ShardingSphere 体验小结
***
## 简介
在前几篇文章中，我们尝试了ShardingSphere JDBC、Proxy、UI，都是源码上直接运行的，跑了起来，看到相应的效果，为后面的源码分析做好准备，今天就简单做一个小结

## 总结
这几天对于ShardingSphere的相关了解如下：


![ShardingSphere源码阅读.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f9ed6777b5d54edf9a7e7321bf0ae462~tplv-k3u1fbpfcp-watermark.image)

目前体验了三大部分：

- ShardingSphere JDBC
- ShardingSphere Proxy
- ShardingSphere UI

JDBC和Proxy的功能基本是相同的，都有数据分片、读写分离、数据加密都功能

不同的是JDBC对于业务代码来说，应该是侵入式的，而Proxy是非侵入式的

但使用Proxy就相当于引入新的部署单元，增加维护成本

UI是体验了Proxy集成部分，能查看Proxy的服务状态和其上的数据分片等的配置信息，还能动态更新配置到Proxy上

示例部分就基本到这了，接下来就是源码分析部分了