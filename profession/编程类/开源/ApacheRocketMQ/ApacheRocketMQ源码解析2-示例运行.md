# ApacheRocketMQ源码解析（二）示例运行
***
## 简介
&ensp;&ensp;&ensp;&ensp;介绍拉取GitHub工程，在本地进行运行，启动一个生成和消费示例，为后面源码分析做准备

## 运行操作日志
### 源码下载
&ensp;&ensp;&ensp;&ensp;首先从GitHub上克隆项目，在本地后使用Maven安装依赖，操作日志如下：

```shell script
git clone https://github.com.cnpmjs.org/apache/rocketmq.git
```

&ensp;&ensp;&ensp;&ensp;使用IDEA的Maven工具即可，跳过测试，再使用阿里云镜像，很快就搞定：[Maven 阿里云仓库使用小技巧](https://juejin.cn/post/6911192947019120654)

&ensp;&ensp;&ensp;&ensp;PS：注意使用Java8，请注意检查IDEA的工程运行版本，不然后面的运行会报错，找不到xxxx

### 运行
&ensp;&ensp;&ensp;&ensp;通过官方文档的阅读，我们知道它有四个主要模块：

- NameServer：类似注册中心，持有所有Broker信息
- BrokerServer：消息存储和操作，核心
- Producer/Consumer：生产者和消费者，用户进行使用的模块

#### NameServer运行
&ensp;&ensp;&ensp;&ensp;直接运行模块：namesrv，下的主函数，会报错，提示需要设置环境变量

&ensp;&ensp;&ensp;&ensp;我们配置运行，在IDEA编辑运行配置，在环境变量中填入：ROCKETMQ_HOME=F:\Code\gitclone\rocketmq\distribution

&ensp;&ensp;&ensp;&ensp;直接设置为我们当前工程下的目录：distribution，里面有所以的配置，不用像其他文章中还建立其他文件

&ensp;&ensp;&ensp;&ensp;图形界面设置参考链接：[启动nameser报错Please set the ROCKETMQ_HOME variable in your environment to match the location of](https://blog.csdn.net/ppwwp/article/details/102652370)

&ensp;&ensp;&ensp;&ensp;修改完成后成功启动：

```text
Connected to the target VM, address: '127.0.0.1:62694', transport: 'socket'
The Name Server boot success. serializeType=JSON
```

#### BrokerServer运行
&ensp;&ensp;&ensp;&ensp;同NameServer，需要设置环境变量，在IDEA中直接配置即可，和NameServer一模一样

&ensp;&ensp;&ensp;&ensp;我们配置运行，在IDEA编辑运行配置，在环境变量中填入：ROCKETMQ_HOME=F:\Code\gitclone\rocketmq\distribution

&ensp;&ensp;&ensp;&ensp;还有设置一个NameServer的地址，看代码好像是通过环境变量设置的，但一直不得行，就直接改代码了，改动如下：

```java
public class BrokerConfig {
    private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.COMMON_LOGGER_NAME);

    private String rocketmqHome = System.getProperty(MixAll.ROCKETMQ_HOME_PROPERTY, System.getenv(MixAll.ROCKETMQ_HOME_ENV));
//    @ImportantField
//    private String namesrvAddr = System.getProperty(MixAll.NAMESRV_ADDR_PROPERTY, System.getenv(MixAll.NAMESRV_ADDR_ENV));
    private String namesrvAddr = "192.168.101.104:9876";
}
```

&ensp;&ensp;&ensp;&ensp;修改完成后启动模块：broker，下的主函数，成功运行

```text
Connected to the target VM, address: '127.0.0.1:50992', transport: 'socket'
The broker[DESKTOP-1JVUVP4, 172.25.144.1:10911] boot success. serializeType=JSON and name server is 192.168.101.104:9876
```

#### Producer运行
&ensp;&ensp;&ensp;&ensp;在模块：example，下有个quicksort目录，我们修改类：Producer，主要是设置NameServer的地址，代码大致如下：

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

&ensp;&ensp;&ensp;&ensp;运行Producer，一大排日志，打印，运行成功：

```text
SendResult [sendStatus=SEND_OK, msgId=7F000001820818B4AAC2931A879E03E6, offsetMsgId=AC19900100002A9F0000000000062F7E, messageQueue=MessageQueue [topic=TopicTest, brokerName=DESKTOP-1JVUVP4, queueId=2], queueOffset=499]
SendResult [sendStatus=SEND_OK, msgId=7F000001820818B4AAC2931A879F03E7, offsetMsgId=AC19900100002A9F0000000000063049, messageQueue=MessageQueue [topic=TopicTest, brokerName=DESKTOP-1JVUVP4, queueId=3], queueOffset=499]
13:33:09.419 [NettyClientSelector_1] INFO  RocketmqRemoting - closeChannel: close the connection to remote address[192.168.101.104:9876] result: true
13:33:09.421 [NettyClientSelector_1] INFO  RocketmqRemoting - closeChannel: close the connection to remote address[172.25.144.1:10911] result: true
Disconnected from the target VM, address: '127.0.0.1:51625', transport: 'socket'
```

#### Consumer运行
&ensp;&ensp;&ensp;&ensp;在模块：example，下有个quicksort目录，我们修改类：Consumer，主要是设置NameServer的地址，代码大致如下：

```java
public class Producer {
    public static void main(String[] args) throws MQClientException, InterruptedException {
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
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

&ensp;&ensp;&ensp;&ensp;运行Consumer，一大排日志进行打印

```text
ConsumeMessageThread_4 Receive New Messages: [MessageExt [brokerName=DESKTOP-1JVUVP4, queueId=3, storeSize=203, queueOffset=237, sysFlag=0, bornTimestamp=1611898133131, bornHost=/172.25.144.1:51072, storeTimestamp=1611898133131, storeHost=/172.25.144.1:10911, msgId=AC19900100002A9F000000000002F019, commitLogOffset=192537, bodyCRC=329761110, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=250, CONSUME_START_TIME=1611898199080, UNIQ_KEY=7F000001084018B4AAC293169E8B03B5, CLUSTER=DefaultCluster, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 32, 57, 52, 57], transactionId='null'}]] 
ConsumeMessageThread_9 Receive New Messages: [MessageExt [brokerName=DESKTOP-1JVUVP4, queueId=3, storeSize=203, queueOffset=236, sysFlag=0, bornTimestamp=1611898133127, bornHost=/172.25.144.1:51072, storeTimestamp=1611898133128, storeHost=/172.25.144.1:10911, msgId=AC19900100002A9F000000000002ECED, commitLogOffset=191725, bodyCRC=437357949, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=250, CONSUME_START_TIME=1611898199080, UNIQ_KEY=7F000001084018B4AAC293169E8703B1, CLUSTER=DefaultCluster, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 32, 57, 52, 53], transactionId='null'}]] 
ConsumeMessageThread_19 Receive New Messages: [MessageExt [brokerName=DESKTOP-1JVUVP4, queueId=3, storeSize=203, queueOffset=235, sysFlag=0, bornTimestamp=1611898133125, bornHost=/172.25.144.1:51072, storeTimestamp=1611898133125, storeHost=/172.25.144.1:10911, msgId=AC19900100002A9F000000000002E9C1, commitLogOffset=190913, bodyCRC=494684516, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=250, CONSUME_START_TIME=1611898199080, UNIQ_KEY=7F000001084018B4AAC293169E8503AD, CLUSTER=DefaultCluster, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 32, 57, 52, 49], transactionId='null'}]] 
ConsumeMessageThread_17 Receive New Messages: [MessageExt [brokerName=DESKTOP-1JVUVP4, queueId=3, storeSize=203, queueOffset=234, sysFlag=0, bornTimestamp=1611898133123, bornHost=/172.25.144.1:51072, storeTimestamp=1611898133123, storeHost=/172.25.144.1:10911, msgId=AC19900100002A9F000000000002E695, commitLogOffset=190101, bodyCRC=996047510, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=250, CONSUME_START_TIME=1611898199080, UNIQ_KEY=7F000001084018B4AAC293169E8303A9, CLUSTER=DefaultCluster, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 32, 57, 51, 55], transactionId='null'}]] 
```

## 总结
&ensp;&ensp;&ensp;&ensp;本篇文章，详细记录了如果使用IDEA运行Apache RocketMQ源码，运行了工程中的一个简单的生产者和消费者示例，为后面的Debug处理流程研究打下基础，后面文章开始研究处理流程

## 源码解析文章目录
### 掘金
#### 示例入门
- [ApacheRocketMQ源码解析（一）概览](https://juejin.cn/post/6923053424300785677/)