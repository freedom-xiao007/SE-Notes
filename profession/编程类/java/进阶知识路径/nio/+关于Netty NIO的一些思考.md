# 关于Netty NIO的一些思考
***
## 简介
介绍Netty NIO网络编程的高性能原理

## 高性能之道
首先明确高性能的切入点：填充I/O操作的CPU空闲时间，提升CPU运行效率，重复使用CPU达到高性能目的

利用下面的三个手段到达高性能目标：

- 1.reactor网络模型
- 2.多线程
- 3.零拷贝:这里就不进行讲解，Netty这块的核心是使用堆外内存，减少拷贝次数

### reactor网络模型和多线程
在单线程的模式下，由于线程进行网络I/O操作会发生阻塞，导致CPU处于空闲

而reactor网络模型加多线程能充分的利用的CPU资源

如下图所示，NIO简化的关键模型如下：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/acfbfb2ab24e466ebf33c76b4a46cdf5~tplv-k3u1fbpfcp-watermark.image))

两个关键地方：

- Boss线程池
- Worker线程池

在Boss线程池中，运行接收请求建立连接工作的线程，这部分工作不多，大部分情况一个线程就够。主要的目的就是不就行逻辑处理，先建立连接，把请求放到后面Worker中

Worker线程池的工作就相对比较多，所有也比较核心

首先明确一个概念：Worker线程池中一个线程会绑定多个Socket，而该线程会轮询绑定的所有Socket，逐一的对读写事件进行操作

比如当前线程绑定了三个Socket，第二三分别来了读写事件：

- Socket1：无事件
- Socket2：读事件
- Socket3：写事件

那线程的工作流程就会如下：

- 轮询到Socket1，发现没有事件，直接跳过
- 轮询到Socket2，发现读事件，读取相应的数据，进行处理（处理不应该阻塞，便于进行下一个Socket的事件处理）
- 上一步处理完成后，轮询Socket3，发现写事件，调用相应的接口，发送数据出去

结合上非阻塞异步编程，发现线程基本上都是在工作的，效率极高

通过上面的分析也发现，在Netty使用中，不要在Handler中使用阻塞操作，耗时的操作请使用多线程进行异步

更多的详情请查看下面的两个参考链接，特别是kafka的，我是看了这篇文章才想通了一直不懂的Worker处理那块

## 参考链接
- [Netty 系列之 Netty 高性能之道](https://www.infoq.cn/article/netty-high-performance)
- [07 | SocketServer（上）：Kafka到底是怎么应用NIO实现网络通信的？](https://time.geekbang.org/column/article/231139)
