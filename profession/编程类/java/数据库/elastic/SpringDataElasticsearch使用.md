# Spring Data Elasticsearch 使用示例

***

一起养成写作习惯！这是我参与「掘金日新计划 · 4 月更文挑战」的第 26 天，[点击查看活动详情](https://juejin.cn/post/7080800226365145118)。

## 简介

在以前的文章中，介绍过如果使用Elasticsearch提供的RestHighClient之类的操作Elasticsearch，本篇将介绍如何使用Spring封装好的Elasticsearch

## Maven 依赖

Spring Data Elasticsearch的依赖如下

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
    <description>Demo project for Spring Boot</description>
    <properties>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
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

## Elasticsearch的model

使用Elasticsearch时，类型Mybatis，我们也需要定义数据模型，让其能对应上Elasticsearch中的数据结构

定义大致如下：

```java
package com.ninetech.cloud.opeate.warn.server.elastic.model;

import com.fasterxml.jackson.annotation.JsonFormat;
import com.fasterxml.jackson.annotation.JsonProperty;
import com.google.gson.annotations.SerializedName;
import lombok.Builder;
import lombok.Data;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.DateFormat;
import org.springframework.data.elasticsearch.annotations.Document;
import org.springframework.data.elasticsearch.annotations.Field;
import org.springframework.data.elasticsearch.annotations.FieldType;

import java.util.Date;

@Data
@Builder
@Document(indexName = "test-index")
public class TestModel {

    @Id
    @Field(type = FieldType.Text)
    private String id;

    @Field(type = FieldType.Text)
    private String name;
}
```

如上所示，我们简单定义了一个，其中的@Id是Elasticsearch文档的唯一标识，如果在保存的时候不填入，则会自动生成

@Field是字段类型，有很多的类型，这方面请参考文档，自行搜索查看

## Repository

同MySQL和MongoDB，我们也需要定义的一个Repository层

```java
public interface TestRepository extends ElasticsearchRepository<TestModel, String> {

    List<TestModel> findByName(final String name);
}
```

如上所示，和Mybatis Plus很像，这里也继承了一个基础的ElasticsearchRepository,其中内置很多的基础常用方法

我们还自定义了一个根据Name查看的方法，这里可以看到MongoDB和Elasticsearch这两个非关系数据的使用上还是挺像的

## Service

数据模型和Repository都准备好了，下面我们来看看如何使用

我们就简单的进行下保存操作和查询操作

```java
@Slf4j
@Service
@AllArgsConstructor
public class TestService {

    private final TestRepository testRepository;

    public void save( final String name) {
        final TestModel log = TestModel.builder()
                .name(name)
                .build();
        testRepository.save(log);
    }

    public List<TestModel> list(final String name) {
	return testRepository.findByName(name);
    }
}
```

如上所示，使用起来还是挺方便的

## 总结

本篇介绍了在Spring Data中Elasticsearch的基本使用，可以看出来和MongoDB差不多，同样也是使用上比较方便

但如果面对复杂查询话，目前使用下来还是有点吃力（也有可能是不够熟系）

## 参考链接

- [9.2.5. Elasticsearch](https://docs.spring.io/spring-boot/docs/2.6.6/reference/htmlsingle/#data.nosql.elasticsearch)