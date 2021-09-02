## 简介
本文主要是对API网关：apache/incubator-shenyu目前的功能和模块做一个简单的介绍

## 功能与模块说明
整个[ShenYu](https://github.com/apache/incubator-shenyu)网关的模块与交互如下：

![ApacheShenYu简介.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/49743f5a2ca8434ab687cd8e4128eb06~tplv-k3u1fbpfcp-watermark.image)

核心点是网关转发部分：Bootstrap网关。它主要负责将外部的请求，转发到对应的后台的服务，目前能支持HTTP和RPC两类的请求转发

如果使用过Nginx，那转发的规则是需要配置的，而ShenYu的：管理后台Admin就是配置规则的地方

目前转发规则的配置方式有两种：

- 手工配置：在后台管理界面，人工的去配置相关的转发规则
- 自动配置：使用ShenYu提供的Client注册相关的依赖，在服务启动时自动进行注册，目前自动注册只支持Java语句的后台服务，其他语言的只能手动配置

自动注册目前提供多种方式：Zookeeper、Etcd、Nacos、HTTP等等，数据也能依赖与注册中心自动更新

当规则等信息注册更新后，就会通过数据同步中心，将最新的规则同步给网关，在Bootstrap网关不进行操作的情况下，能适应新的规则

当网关Bootstrap中，不止用于能进行转发，其支持其他的如限流熔断、黑名单、加解密等一些功能，提供了比较灵活的扩展机制，可以根据自己的需求定制化开发网关插件