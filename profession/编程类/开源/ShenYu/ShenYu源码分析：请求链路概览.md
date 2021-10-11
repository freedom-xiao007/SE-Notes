# ShenYu源码分析：请求链路概览
***
## 简介
本篇文章中，将分析ShenYu网关中处理一个请求的完整路径，让大家对ShenYu的核心处理部分有个大致的了解



## 环境准备与示例运行
本文基于最新的主线代码进行分析：[https://github.com/apache/incubator-shenyu](https://github.com/apache/incubator-shenyu)

只需运行下面三个组件即可：shenyu-admin、shenyu-bootstrap、shenyu-examples-http

- shenyu-admin : 规则配置等管理后台，可手动和自动新增更新路由相关的规则数据
- shenyu-bootstrap ： 网关核心，处理请求，转发到相关的后台服务
- shenyu-examples-http ： 官方的HTTP示例工程

shenyu-admin 需要手工修改处理MySQL相关的部分即可，其他的直接运行即可，不用进行修改

查看shenyu-examples-http的Controller，找一个存在的路由进行访问：



```shell script
# linux下的命令
curl --request GET \
  --url 'http://127.0.0.1:9195/http/test/findByUserId?userId=1' \
  --header 'cache-control: no-cache' \
  --header 'postman-token: caf06a72-772f-8b30-04db-9710729125b6'
  
# Windows下的命令，使用powershell
curl --request GET `
  --url 'http://127.0.0.1:9195/http/test/findByUserId?userId=1' `
  --header 'cache-control: no-cache' `
  --header 'postman-token: caf06a72-772f-8b30-04db-9710729125b6'
  
# 得到返回的结果
{"userId":"1","userName":"hello world"}
```



更多的详情的内容可自行查询官方文档：[Http快速开始](https://shenyu.apache.org/zh/docs/quick-start/quick-start-http)



## 源码分析

### 切入点寻找

在阅读源码的过程中，可以从遇到问题点处、链路终点处、通过官方资料等得到核心代码处等打上断点，跟踪调用栈去分析代码得到我们想要的东西

ShenYu不熟悉的话还真不好打断点，但我们在目前发起上面的请求时，Bootstrap的日志都会有下面的输出：

```tex
org.apache.shenyu.plugin.base.AbstractShenyuPlugin - context_path selector success match , selector name :/context-path/http
org.apache.shenyu.plugin.base.AbstractShenyuPlugin - context_path rule success match , rule name :/context-path/http
org.apache.shenyu.plugin.base.AbstractShenyuPlugin - divide selector success match , selector name :/http
org.apache.shenyu.plugin.base.AbstractShenyuPlugin - divide rule success match , rule name :/http/test/**
```



在上面的日志中，看到匹配和我们请求路径相关的东西，直接全局搜索定位到相关的代码打上断点
