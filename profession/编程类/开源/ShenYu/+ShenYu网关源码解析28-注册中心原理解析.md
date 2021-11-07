# ShenYu网关源码解析--注册中心原理解析
***

## 简介
本文介绍ShenYu网关中的注册中心模块的原理及设计，基于ShenYu网关版本：V2.4.1

## 概览
作为一个API网关，动态的路由配置是一个很重要的功能了，下图是目前ShenYu网关的路由配置界面：

![HTTP1](./picture/HTTP1.png)

![HTTP2](./picture/HTTP2.png)

上图是HTTP请求的代理路由配置显示界面，通过客户端自动注入或者手动输入。第一张图中右部分是转发匹配规则，请求到来时会根据上面的路径进行匹配

第二张是集群配置相关的，用于负载均衡，HTTP请求类型在ShenYu中需要配置集群列表，图中的处理处便是填个集群中的各个机器

当然不是所有的请求类型都需要配置集群列表，像Dubbo、SpringCloud就不需要，因为Dubbo客户端之类自己有相应的负载均衡功能，就不需要在ShenYu中进行配置

下图便是SpringCloud的配置界面，第二张图中可以看到，不需要配置集群列表

![SpringCloud1](./picture/SpringCloud1.png)

![SpringCloud2](./picture/SpringCloud2.png)

首先明确注册中心主要注册的是什么：

- 匹配转发规则：根据规则转发请求到对应的服务器
- 集群列表：需要ShenYu做负载均衡的请求类型需要配置

## 注册中心的设计

*也可参考官方文档：https://dromara.org/zh/projects/soul/register-center-design/*

