# RabbitMQ
***

## 原生使用

```java
import com.rabbitmq.client.*;
import lombok.SneakyThrows;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.concurrent.BasicThreadFactory;

import java.io.IOException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Timer;
import java.util.TimerTask;
import java.util.concurrent.*;

@Slf4j
public class RabbitMqTest {
    
    private RabbitConfig config;
    private Connection conn;
    private Channel channel;
    private final String exchangeName = "direct";
    String queueName;

    private void connectRabbitmq(final String uuid) throws IOException, TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setUsername(config.getUsername());
        factory.setPassword(config.getPassword());
        factory.setVirtualHost(config.getBackendVhost());
        factory.setHost(config.getHost());
        factory.setPort(config.getPort());
        conn = factory.newConnection();
        channel = conn.createChannel();

        channel.exchangeDeclare(exchangeName, "direct", true);
        channel.queueDeclare(queueName, true, false, false, null);
        channel.queueBind(queueName, exchangeName, queueName);

        channel.basicConsume(queueName, true, "myConsumerTag",
                new DefaultConsumer(channel) {
                    @Override
                    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                                               byte[] body) throws IOException {
                        final String msg = new String(body);
                        log.info("msg: {}", msg);
                    }
                });
    }

    private void publish() throws IOException {
        final String queueName =  "queue";
        byte[] messageBodyBytes = "message".getBytes();
        task.basicPublish(exchangeName, queueName, null, messageBodyBytes);
        log.info("发布消息");
    }

    public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
        RabbitConfig config = RabbitConfig.builder()
                .backendVhost("/")
                .host("xxx.xxx.xxx.xxx")
                .password("guest")
                .port(5672)
                .username("guest")
                .virtualHost("/")
                .build();
    }
}
```

## 参考链接
### 教程
- [Spring RabbitMQ官方教程](https://docs.spring.io/spring-amqp/docs/current/reference/html/)
- [SpringBoot RabbitMQ配置多vhost/多RabbitMQ实例方案_雨潇的专栏-程序员宅基地](https://www.cxyzjd.com/article/yuxiao97/111642760)
- [Java Client API Guide](https://www.rabbitmq.com/api-guide.html)