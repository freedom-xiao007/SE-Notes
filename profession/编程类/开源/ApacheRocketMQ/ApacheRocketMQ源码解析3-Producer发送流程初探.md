# ApacheRocketMQ源码解析（三）Producer发送流程初探
***
## 简介
&ensp;&ensp;&ensp;&ensp;开始进入源码分析，本篇文章粗略看下Producer发送消息大致的处理路径是什么样的，为后面的分析打基础

## 概览
&ensp;&ensp;&ensp;&ensp;代码运行请参考之前的文章：[ApacheRocketMQ源码解析（二）示例运行](https://juejin.cn/post/6923054258656903182/)

&ensp;&ensp;&ensp;&ensp;我们从Producer示例的发送函数开始跟踪调用，看看其中经历了那些重要的类，进行了那些处理，最终跟踪到netty的发送函数结束

## 源码Debug
&ensp;&ensp;&ensp;&ensp;PS:在Debug的时候会稍微停留下就会超时，在超时的函数打上断点，点击IDEA左上加的绿色三角形，快速再次跳到超时的位置进入即可

&ensp;&ensp;&ensp;&ensp;我们在下面的示例代码中：SendResult sendResult = producer.send(msg);打上断点，进行调用用跟踪

```java
public class Producer {
    public static void main(String[] args) throws MQClientException, InterruptedException {
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
        // 主要是增加这么一句，设置NameServer地址
        producer.setNamesrvAddr("192.168.101.104:9876");
        producer.start();

        for (int i = 0; i < 1000; i++) {
            try {
                Message msg = new Message("TopicTest" /* Topic */,
                    "TagA" /* Tag */,
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
                );
                SendResult sendResult = producer.send(msg);
                System.out.printf("%s%n", sendResult);
            } catch (Exception e) {
                e.printStackTrace();
                Thread.sleep(1000);
            }
        }
        producer.shutdown();
    }
}
```

&ensp;&ensp;&ensp;&ensp;不断的转折来到下面的函数，通过调试，发现这个操作里面大部分是时间相关的操作，并且得到了Broker的地址信息

```java
public class DefaultMQProducerImpl implements MQProducerInner {
    private SendResult sendDefaultImpl(
        Message msg,
        final CommunicationMode communicationMode,
        final SendCallback sendCallback,
        final long timeout
    ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        this.makeSureStateOK();
        Validators.checkMessage(msg, this.defaultMQProducer);
        final long invokeID = random.nextLong();
        long beginTimestampFirst = System.currentTimeMillis();
        long beginTimestampPrev = beginTimestampFirst;
        long endTimestamp = beginTimestampFirst;
        TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
        ......
        if (topicPublishInfo != null && topicPublishInfo.ok()) {
            String[] brokersSent = new String[timesTotal];
            for (; times < timesTotal; times++) {
                String lastBrokerName = null == mq ? null : mq.getBrokerName();
                // 得到Broker相关的信息：MessageQueue [topic=TopicTest, brokerName=DESKTOP-1JVUVP4, queueId=0]
                MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
                if (mqSelected != null) {
                    mq = mqSelected;
                    brokersSent[times] = mq.getBrokerName();
                    try {
                        .......
                        // 发送消息
                        sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
                        endTimestamp = System.currentTimeMillis();
                        .......
                } else {
                    break;
                }
            }
            .......
    }
}
```

&ensp;&ensp;&ensp;&ensp;继续跟踪发送消息的函数，来到同一个类的下面一个函数

&ensp;&ensp;&ensp;&ensp;下面的函数感觉大体是设置请求头属性相关的信息，因为有大量的set之类的操作

```java
public class DefaultMQProducerImpl implements MQProducerInner {
    private SendResult sendKernelImpl(final Message msg,
        final MessageQueue mq,
        final CommunicationMode communicationMode,
        final SendCallback sendCallback,
        final TopicPublishInfo topicPublishInfo,
        final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        long beginStartTime = System.currentTimeMillis();
        String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
        if (null == brokerAddr) {
            tryToFindTopicPublishInfo(mq.getTopic());
            brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
        }

        SendMessageContext context = null;
        if (brokerAddr != null) {
            // 得到VIP的Broker的IP和端口地址，VIP好像是Broker运行打印的，这个有点意思，后面再看看
            brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);
            byte[] prevBody = msg.getBody();
            try {
                ......
                SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
                requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
                requestHeader.setTopic(msg.getTopic());
                requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
                requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
                requestHeader.setQueueId(mq.getQueueId());
                requestHeader.setSysFlag(sysFlag);
                requestHeader.setBornTimestamp(System.currentTimeMillis());
                requestHeader.setFlag(msg.getFlag());
                requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
                requestHeader.setReconsumeTimes(0);
                requestHeader.setUnitMode(this.isUnitMode());
                requestHeader.setBatch(msg instanceof MessageBatch);
                ......
                SendResult sendResult = null;
                switch (communicationMode) {
                    .....
                    case SYNC:
                        long costTimeSync = System.currentTimeMillis() - beginStartTime;
                        if (timeout < costTimeSync) {
                            throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
                        }
                        // send message 发送消息
                        sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                            brokerAddr,
                            mq.getBrokerName(),
                            msg,
                            requestHeader,
                            timeout - costTimeSync,
                            communicationMode,
                            context,
                            this);
                        break;
                    default:
                        assert false;
                        break;
                }
                ......
    }
}
```

&ensp;&ensp;&ensp;&ensp;再跟消息发送的函数，来到下面的代码

&ensp;&ensp;&ensp;&ensp;大部分代码没有看懂，但看到一个重要的设置Body

```java
public class MQClientAPIImpl {
    public SendResult sendMessage(
        ......
    ) throws RemotingException, MQBrokerException, InterruptedException {
        long beginStartTime = System.currentTimeMillis();
        RemotingCommand request = null;
        String msgType = msg.getProperty(MessageConst.PROPERTY_MESSAGE_TYPE);
        boolean isReply = msgType != null && msgType.equals(MixAll.REPLY_MESSAGE_FLAG);
        if (isReply) {
            if (sendSmartMsg) {
                SendMessageRequestHeaderV2 requestHeaderV2 = SendMessageRequestHeaderV2.createSendMessageRequestHeaderV2(requestHeader);
                request = RemotingCommand.createRequestCommand(RequestCode.SEND_REPLY_MESSAGE_V2, requestHeaderV2);
            } else {
                request = RemotingCommand.createRequestCommand(RequestCode.SEND_REPLY_MESSAGE, requestHeader);
            }
        } else {
            if (sendSmartMsg || msg instanceof MessageBatch) {
                SendMessageRequestHeaderV2 requestHeaderV2 = SendMessageRequestHeaderV2.createSendMessageRequestHeaderV2(requestHeader);
                request = RemotingCommand.createRequestCommand(msg instanceof MessageBatch ? RequestCode.SEND_BATCH_MESSAGE : RequestCode.SEND_MESSAGE_V2, requestHeaderV2);
            } else {
                request = RemotingCommand.createRequestCommand(RequestCode.SEND_MESSAGE, requestHeader);
            }
        }
        // 前面的没看懂，就看懂了一个setbody
        request.setBody(msg.getBody());

        switch (communicationMode) {
            ......
            case SYNC:
                long costTimeSync = System.currentTimeMillis() - beginStartTime;
                if (timeoutMillis < costTimeSync) {
                    throw new RemotingTooMuchRequestException("sendMessage call timeout");
                }
                // send message 消息发送
                return this.sendMessageSync(addr, brokerName, msg, timeoutMillis - costTimeSync, request);
            default:
                assert false;
                break;
        }

        return null;
    }
}
```

&ensp;&ensp;&ensp;&ensp;继续跟发送消息的函数，来到了官方文档中一个提过的比较重要的通信的类，代码如下：

&ensp;&ensp;&ensp;&ensp;内容好像不是太多，看不出比较重要的东西

```java
public class NettyRemotingClient extends NettyRemotingAbstract implements RemotingClient {
    public RemotingCommand invokeSync(String addr, final RemotingCommand request, long timeoutMillis)
        throws InterruptedException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException {
        long beginStartTime = System.currentTimeMillis();
        final Channel channel = this.getAndCreateChannel(addr);
        if (channel != null && channel.isActive()) {
            try {
                doBeforeRpcHooks(addr, request);
                long costTime = System.currentTimeMillis() - beginStartTime;
                if (timeoutMillis < costTime) {
                    throw new RemotingTimeoutException("invokeSync call timeout");
                }
                // 前面也不知道在干啥，只知道这个发送请求
                RemotingCommand response = this.invokeSyncImpl(channel, request, timeoutMillis - costTime);
                doAfterRpcHooks(RemotingHelper.parseChannelRemoteAddr(channel), request, response);
                return response;
            } catch (RemotingSendRequestException e) {
                log.warn("invokeSync: send request exception, so close the channel[{}]", addr);
                this.closeChannel(addr, channel);
                throw e;
            } catch (RemotingTimeoutException e) {
                if (nettyClientConfig.isClientCloseSocketIfTimeout()) {
                    this.closeChannel(addr, channel);
                    log.warn("invokeSync: close socket because of timeout, {}ms, {}", timeoutMillis, addr);
                }
                log.warn("invokeSync: wait response timeout exception, the channel[{}]", addr);
                throw e;
            }
        } else {
            this.closeChannel(addr, channel);
            throw new RemotingConnectException(addr);
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;继续跟着消息发送，来到了下面的终点代码

&ensp;&ensp;&ensp;&ensp;还好Netty用过，能看懂个大概，但细节部分不是太懂。刚好MQ Demo也有用netty作为通信方式，RocketMQ没有用响应式的方式，这个感觉回过头来研究下请求和结果的获取

```java
public abstract class NettyRemotingAbstract {
    public RemotingCommand invokeSyncImpl(final Channel channel, final RemotingCommand request,
        final long timeoutMillis)
        throws InterruptedException, RemotingSendRequestException, RemotingTimeoutException {
        final int opaque = request.getOpaque();

        try {
            final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis, null, null);
            this.responseTable.put(opaque, responseFuture);
            final SocketAddress addr = channel.remoteAddress();
            // 下面的函数，发送请求到Broker中，并得到了结果
            channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture f) throws Exception {
                    if (f.isSuccess()) {
                        responseFuture.setSendRequestOK(true);
                        return;
                    } else {
                        responseFuture.setSendRequestOK(false);
                    }

                    responseTable.remove(opaque);
                    responseFuture.setCause(f.cause());
                    responseFuture.putResponse(null);
                    log.warn("send a request command to channel <" + addr + "> failed.");
                }
            });

            RemotingCommand responseCommand = responseFuture.waitResponse(timeoutMillis);
            if (null == responseCommand) {
                if (responseFuture.isSendRequestOK()) {
                    throw new RemotingTimeoutException(RemotingHelper.parseSocketAddressAddr(addr), timeoutMillis,
                        responseFuture.getCause());
                } else {
                    throw new RemotingSendRequestException(RemotingHelper.parseSocketAddressAddr(addr), responseFuture.getCause());
                }
            }
            // 这个是结果？
            return responseCommand;
        } finally {
            this.responseTable.remove(opaque);
        }
    }
}
```

## 总结
&ensp;&ensp;&ensp;&ensp;本篇文章就大致跟踪了下大致的Producer的发送流程，没有做分析（因为很多看不懂），这里发现代码比之前看的网关源码要复杂的多

&ensp;&ensp;&ensp;&ensp;这里先大致梳理下流程：

- Producer : 生产者send函数
- DefaultMQProducerImpl ：构造请求，设置时间相关、请求头等等；好像还有重试，默认为3次
- MQClientAPIImpl : 设置请求的body
- NettyRemotingClient : 发送请求到Broker，拿到结果

&ensp;&ensp;&ensp;&ensp;在知道大致流程的情况下，可进一步研究的细节有：

- RocketMQ的Netty如果取得和处理得到的响应的？

## 源码解析文章目录
### 掘金
#### 示例入门
- [ApacheRocketMQ源码解析（一）概览](https://juejin.cn/post/6923053424300785677/)
- [ApacheRocketMQ源码解析（二）示例运行](https://juejin.cn/post/6923054258656903182/)
