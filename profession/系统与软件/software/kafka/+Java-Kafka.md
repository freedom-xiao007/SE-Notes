# Java Kafka 简单示例
***
## 简介
&ensp;&ensp;&ensp;&ensp;Java kafka 简单代码示例

### maven依赖配置
```java
<!-- kafka -->
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>0.11.0.0</version>
</dependency>
```

### kakfa生产和消费者生成
```java
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.util.*;

/**
 * @author lw1243925457
 */
public class KafkaUtil {

    public static KafkaConsumer<String, String> createConsumer(String servers, String topic) {
        Properties properties = new Properties();
        properties.put("bootstrap.servers", servers);
        properties.put("group.id", "group-1");
        properties.put("enable.auto.commit", "false");
        properties.put("auto.commit.interval.ms", "1000");
        properties.put("auto.offset.reset", "earliest");
        properties.put("session.timeout.ms", "30000");
        properties.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<String, String>(properties);
        kafkaConsumer.subscribe(Arrays.asList(topic));
        return kafkaConsumer;
    }

    public static void readMessage(KafkaConsumer<String, String> kafkaConsumer, int timeout) {
        while (true) {
            ConsumerRecords<String, String> records = kafkaConsumer.poll(timeout);
            for (ConsumerRecord<String, String> record : records) {
                String value = record.value();
                kafkaConsumer.commitAsync();
                System.out.println(value);
            }
        }
    }

    public static KafkaProducer<String, String> createProducer(String servers) {
        Properties properties = new Properties();
        properties.put("bootstrap.servers", servers);
        properties.put("acks", "all");
        properties.put("retries", 0);
        properties.put("batch.size", 16384);
        properties.put("linger.ms", 1);
        properties.put("buffer.memory", 33554432);
        properties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        return new KafkaProducer<String, String>(properties);
    }

    public static void send(KafkaProducer<String, String> producer, String topic, String message) {
        producer.send(new ProducerRecord<String, String>(topic, message));
    }
}
```

### 运行

```java
public class Main {

    public static void main(String[] args) {
        String servers = "localhost:9092,localhost:9093,localhost:9094";
        String topic = "TestTopic";
        String message = "test";

        KafkaProducer<String, String> producer = KafkaUtil.createProducer(servers);
        KafkaUtil.send(producer, topic, message);

        KafkaConsumer<String, String> consumer = KafkaUtil.createConsumer(servers, topic);
        KafkaUtil.readMessage(consumer, 100);
    }
}
```

## 使用心得
### 总是读取最老的消息
&ensp;&ensp;&ensp;&ensp;可能是group-id的问题，新起一个group-id名称

- earliest:当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，从头开始消费
- latest:当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，消费新产生的该分区下的数据
- none:topic各分区都存在已提交的offset时，从offset后开始消费；只要有一个分区不存在已提交的offset，则抛出异常

## 参考链接
- [java 实现kafka消息生产者和消费者](https://blog.csdn.net/beExcellentOne/article/details/53641953)
- [kafka（三）—Kafka的Java代码示例和配置说明](https://segmentfault.com/a/1190000015886487)
- [Kafka - 偏移量提交](https://blog.csdn.net/u011669700/article/details/80053313)
- [Kafka系列（四）Kafka消费者：从Kafka中读取数据](http://www.dengshenyu.com/%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F/2017/11/14/kafka-consumer.html)  
- [Kafka auto.offset.reset值详解](https://blog.csdn.net/lishuangzhe7047/article/details/74530417)