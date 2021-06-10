# Windows10 Kafka Docker 集群搭建
***
## 简介
&ensp;&ensp;&ensp;&ensp;使用 Windows Docker Desktop 搭建 Kafka 集群

### 运行 Zookeeper
&ensp;&ensp;&ensp;&ensp;这里使用但 zk，使用docker启动即可

```shell script
# 第一次启动
docker run -dit --name zk -p 2181:2181 zookeeper

# 重启
docker restart zk

# 查看日志
docker logs -f zk
```

### 运行 Kafka
&ensp;&ensp;&ensp;&ensp;启动的命令如下，注意将下面的 192.168.101.104 换成自己的宿主机IP，运行后查看日志正常即可

```shell script
# 第一次启动
docker run -dit --name kafka0 -p 9092:9092 -e KAFKA_BROKER_ID=0 -e KAFKA_ZOOKEEPER_CONNECT=192.168.101.104:2181 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://192.168.101.104:9092 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 -t wurstmeister/kafka

docker run -dit --name kafka1 -p 9093:9093 -e KAFKA_BROKER_ID=1 -e KAFKA_ZOOKEEPER_CONNECT=192.168.101.104:2181 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://192.168.101.104:9093 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9093 -t wurstmeister/kafka

docker run -dit --name kafka2 -p 9094:9094 -e KAFKA_BROKER_ID=2 -e KAFKA_ZOOKEEPER_CONNECT=192.168.101.104:2181 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://192.168.101.104:9094 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9094 -t wurstmeister/kafka

# 重启
docker restart kafka0
docker restart kafka1
docker restart kafka2

# 查看日志
docker logs -f kafka0

# 删除kafka
docker rm -f kafka0
docker rm -f kafka1
docker rm -f kafka2
```

### 测试
&ensp;&ensp;&ensp;&ensp;测试建立3的副本和5的partition，查看是否配置成功。然后在1和2上启动消费者，0生产消息

```shell script
# 建立副本和partition
docker exec -ti kafka0 kafka-topics.sh --create --zookeeper 192.168.101.104:2181 --replication-factor 3 --partitions 5 --topic TestTopic
# 查看信息
docker exec -ti kafka0 kafka-topics.sh --describe --zookeeper 192.168.101.104:2181 --topic TestTopic
docker exec -ti kafka1 kafka-topics.sh --describe --zookeeper 192.168.101.104:2181 --topic TestTopic
docker exec -ti kafka2 kafka-topics.sh --describe --zookeeper 192.168.101.104:2181 --topic TestTopic

# 消费和生产，最后一个kafka0输出后在其他两个能看到
docker exec -ti kafka1 kafka-console-consumer.sh --bootstrap-server 192.168.101.104:9093 --topic TestTopic --from-beginning
docker exec -ti kafka2 kafka-console-consumer.sh --bootstrap-server 192.168.101.104:9094 --topic TestTopic --from-beginning
docker exec -ti kafka0 kafka-console-producer.sh --broker-list 192.168.101.104:9092 --topic TestTopic

# 性能测试
docker exec -ti kafka0 kafka-producer-perf-test.sh --topic TestTopic --num-records 100000 --record-size 1000 --throughput 2000 --producer-props bootstrap.servers=192.168.101.104:9092
docker exec -ti kafka0 kafka-consumer-perf-test.sh --bootstrap-server 192.168.101.104:9092 --topic TestTopic --fetch-size 1048576 --messages 100000 --threads 1
```

### kafka manage
&ensp;&ensp;&ensp;&ensp;使用docker启动后，访问: http://localhost:9000/ , 点击添加cluster，输入前两个（名称和zk地址），保存即可

```shell script
docker run -dit -p 9000:9000 -e ZK_HOSTS="192.168.101.104:2181" hlebalbau/kafka-manager:stable

http://localhost:9000/
```

## 参考链接
- [【Kafka精进系列003】Docker环境下搭建Kafka集群](https://blog.csdn.net/noaman_wgs/article/details/103757791)
- [kafka如何彻底删除topic及数据](https://blog.csdn.net/belalds/article/details/80575751)