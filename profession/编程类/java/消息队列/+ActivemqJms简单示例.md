# Activemq Jms 简单示例
***
## 简介
&ensp;&ensp;&ensp;&ensp;简单的 Activemp JMS 示例代码

### activemq 运行
&ensp;&ensp;&ensp;&ensp;简单使用docker启动一个：

```shell script
docker run -dit --name mq -p 11616:61616 -p 8161:8161 rmohr/activemq
```

### maven依赖配置
&ensp;&ensp;&ensp;&ensp;依赖大致如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.4.1</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example.jms.activemq</groupId>
    <artifactId>jms-activemp</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>jms-activemp</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-activemq</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

### 配置类编写
&ensp;&ensp;&ensp;&ensp;配置activemq相关连接，大致如下：

```java
@Configuration
public class JmsConfig {

    private final String BROKER_URL = "tcp://localhost:11616";
    private final String broker_username = "admin";
    private final String broker_password = "admin";

    @Bean
    public ActiveMQConnectionFactory connectionFactory() {
        ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory();
        connectionFactory.setBrokerURL(BROKER_URL);
        connectionFactory.setUserName(broker_username);
        connectionFactory.setPassword(broker_password);
        return connectionFactory;
    }

    @Bean
    public JmsTemplate jmsTemplate() {
        JmsTemplate template = new JmsTemplate();
        template.setConnectionFactory(connectionFactory());
        return template;
    }

    @Bean
    public DefaultJmsListenerContainerFactory jmsListenerContainerFactory() {
        DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory());
        factory.setConcurrency("1-1");
        return factory;
    }
}
```

### 消费监听
&ensp;&ensp;&ensp;&ensp;配置Activemq的监听消费,监听函数有返回值的请参考后面的链接

```java
@Component
public class JmsConsumer {

    @JmsListener(destination = "activeTest")
    public void receiveMessage(final Map message) {
        System.out.println(message.toString());
    }
}
```

### 生产者
&ensp;&ensp;&ensp;&ensp;发送消息到activemq

```java
@Component
public class JmsProducer {

    @Autowired
    private JmsTemplate jmsTemplate;

    public void sendMessage(final String topic, final String message) {
        Map map = new Gson().fromJson(message, Map.class);
        jmsTemplate.convertAndSend(topic, map);
    }
}
```

### 测试运行
&ensp;&ensp;&ensp;&ensp;在主函数发送

```java
@SpringBootApplication
@EnableJms
@Slf4j
public class JmsActivempApplication implements ApplicationRunner {

    @Autowired
    private JmsProducer producer;

    public static void main(String[] args) {
        SpringApplication.run(JmsActivempApplication.class, args);
    }

    @Override
    public void run(ApplicationArguments args) {
        String topic = "activeTest";
        Map<String, String> message = new HashMap<>(1);
        message.put("test", "test");
        log.info("send message to topic " + topic + " :: " + message);
        producer.sendMessage(topic, message);
    }
}
```

## 参考链接
- [Spring Boot ActiveMQ Support](https://www.devglan.com/spring-boot/spring-boot-jms-activemq-example)