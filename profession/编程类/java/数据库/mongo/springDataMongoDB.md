# Spring Data MongoDB 使用示例

***

一起养成写作习惯！这是我参与「掘金日新计划 · 4 月更文挑战」的第 25 天，[点击查看活动详情](https://juejin.cn/post/7080800226365145118)。

## 简介

在以前的文章中，介绍了在Java中如何原生的使用MongoDB，本篇将介绍如果使用Spring封装好的MongoDB Data

## Maven依赖

我们使用一个简单的Web服务作为示例，完整的Maven配置如下：

我们导入MongoDB核心：spring-boot-starter-data-mongodb

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.6.6</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.self.growth</groupId>
    <artifactId>task</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>task</name>
    <description>task</description>
    <properties>
        <java.version>17</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

## 实体/模型和Repository

如同使用mybatis一样，我们也需要创建一个映射到数据库中数据模型的类

如下所示，我们就使用注解标注了ID字段，在MongoDB中，自生成的ID是String类型的

```java
public class TaskConfig {

    @Id
    private String id;
    private String name;
    private String description;
    private LabelEnum label;
    private TaskCycleEnum cycleType;
    private TaskLearnTypeEnum learnType;
    private String group;
    private TaskTypeEnum taskTypeEnum;
    private Boolean isComplete;
    private Date completeDate;
}
```

然后创建一个Mapper之类的Repository层的接口类，基本功能和Mapper差不多

和Mybatis plus很像，其中默认封装了很多的基础查询，一些特殊的查询，在接口中自行添加即可

```java
public interface TaskRepository extends MongoRepository<TaskConfig, String> {

    TaskConfig findByGroupAndName(final String group, final String name);
}
```

如果所示，我们添加了一个方法，是应该数据的group和name字段进行查看的查询函数，并且只返回一条数据

## Controller层与Service层

我们定义一个简单的Controller层如下：一个简单的更新接口和查询列表接口

```java
@RestController
@RequestMapping("/task")
public class TaskController {

    private final TaskService taskService;

    public TaskController(final TaskService taskService) {
        this.taskService = taskService;
    }

    @PostMapping("/update")
    public void update(@RequestBody TaskConfig config) {
        taskService.update(config);
    }

    @GetMapping("/list")
    public List<TaskConfig> list() {
        return taskService.list();
    }
}
```

具体的Service实现如下：

```java
@Service
public record TaskService(TaskRepository taskRepository) {

    public void update(final TaskConfig config) {
        final TaskConfig task = taskRepository.findByGroupAndName(config.getGroup(), config.getName());
        if (task == null) {
            taskRepository.insert(config);
            return;
        }
        taskRepository.delete(task);
        taskRepository.insert(config);
    }

    public List<TaskConfig> list() {
        return taskRepository.findAll();
    }
}
```

目前尝鲜使用的是Java17，体验了一下一些新特性，在上面我们看到在类声明中使用了record关键字，直接在类声明中就传入构造函数所需要的参数

感觉目前编程语言的发展趋势是越来越便捷了，目前感觉还不错

在更新接口中，看到我们先去查看数据库中有没有这条数据，没有则插入，有则先删除，再插入

而列表接口，直接一个findAll返回所有数据

## 总结

我们可以看到很多的查询都是内置的，而且我们目前也没有变成任何的复杂的Sql

spring提供的这些还是一贯的有助于提升开发效率,在声明特有的语句时，之前用过原生的MongoDB查询，就能大致的知道如果使用

- [MongoDB Java 原生使用示例](https://juejin.cn/post/7088668995930292261)