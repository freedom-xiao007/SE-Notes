# Feign 共享登录信息进行请求

***



持续创作，加速成长！这是我参与「掘金日新计划 · 6 月更文挑战」的第2天，[点击查看活动详情](https://juejin.cn/post/7099702781094674468?utm_source=xitongxiaoxi&utm_medium=push&utm_campaign=kechengfenxiao)



## 简介

在开发和一些集成测试中，请求调用需要基于登录，在请求中需要携带登录后得到的token等信息，本篇文章对于这种场景进行了探索



## 背景信息说明

本地实验的有三个组成部分：

- 登录服务：提供用户登录等服务，调用登录接口后，得到后序的token信息
- 业务服务：业务的接口服务，访问接口都需要进行登录验证
- 测试服务：可以当成一个集成测试工程，首先访问登录服务进行登录，得到token信息，然后去访问业务服务

登录服务和业务服务，本篇中不进行说明，本篇中的登录服务和业务服务的登录认证是基于SaToken进行搭建的，可以参考博主之前的一篇文章：[Sa-Token 单点登录 SSO模式二 URL重定向传播会话示例](https://juejin.cn/post/7102733249088077854)

闲话不多说，下面开始测试服务的代码说明



## Maven配置信息

工程中使用spring基础和feign，是一个单独的工程，配置如下：

```xm
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.0</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <groupId>com.self.growth</groupId>
    <artifactId>integration-test</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>integration-test</name>
    <description>integration-test</description>

    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
            <version>3.1.2</version>
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



## 工程配置信息

我们需要进行一些配置，如下：

```yam
test:
  # 登录服务的用户名和密码
  login:
    username: username
    password: password
  server:
    # 业务服务的请求地址
    record:
      url: http://localhost:9050
    # 登录服务的请求地址
    auth:
      url: http://localhost:9000

spring:
  main:
    allow-bean-definition-overriding: true
```



## Feign客户端与拦截器配置

登录服务的客户端定义如下：

```java
package com.self.growth.integration.test.feign;

import com.self.growth.integration.test.vo.ResResult;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient(
        value = "UserClient",
        url = "${test.server.auth.url}"
)
public interface UserClient {

    @RequestMapping(method = RequestMethod.GET, value = "/sso/doLogin")
    ResponseEntity<ResResult<Void>> login(@RequestParam("name") String name, @RequestParam("pwd") String pwd);
}
```

上面定义了客户端的请求url，和一个登录请求，其中使用了ResponseEntity，使用这个能拿到返回的header信息，而登录后的token是返回到header里面的，这个在后面的登录服务有更详细的说明



这里需要一个登录Service，处理得到的请求，保存token信息

```java
package com.self.growth.integration.test.service;

import com.self.growth.integration.test.config.SaTokenContext;
import com.self.growth.integration.test.feign.UserClient;
import com.self.growth.integration.test.vo.ResResult;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;

import java.util.Objects;

@Slf4j
@Service
public class UserService {

    private final UserClient userClient;
    private final SaTokenContext saTokenContext;
    @Value("${test.login.username}")
    private String username;
    @Value("${test.login.password}")
    private String password;

    private boolean isLogin = false;

    public UserService(UserClient userClient, SaTokenContext saTokenContext) {
        this.userClient = userClient;
        this.saTokenContext = saTokenContext;
    }

    public void login() {
        if (!isLogin) {
            ResponseEntity<ResResult<Void>> res = userClient.login(username, password);
            if (Objects.requireNonNull(res.getBody()).getCode() != 200) {
                throw new RuntimeException("用户登录失败");
            }
            isLogin = true;
            log.info("登录成功");
            // 将登录后的token进行保存
            saTokenContext.refreshToken(res.getHeaders());
        }
    }
}
```

登录的逻辑应该是简单明了，saTokenContext可以看出是一个单例，用来保存和提供Token信息，具体代码如下：

```java
package com.self.growth.integration.test.config;

import org.springframework.http.HttpHeaders;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class SaTokenContext {

    private String token;
    private String key;

    /**
     * 从登录返回结果中获取Token信息
     * 
     * 基于SaToken登录认证框架，针对其返回特定进行处理提前
     * @param headers 登录返回的headers
     */
    public void refreshToken(final HttpHeaders headers) {
        final List<String> setCookie = headers.get("set-cookie");
        assert setCookie != null;
        if (setCookie.isEmpty()) {
            return;
        }
        final String originCookie = setCookie.get(0);
        key = originCookie.split(";")[0].split("=")[0];
        token = originCookie.split(";")[0].split("=")[1];
    }

    public String getToken() {
        return token;
    }

    public String getKey() {
        return key;
    }
}
```



在上面登录后，我们对Token进行了保存，token的用途就是在后面的请求中，添加到请求头中

我们这里采用全局拦截处理的方式：将登录后的token放到请求头中。拦截器如下：

```java
package com.self.growth.integration.test.config;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.springframework.cloud.openfeign.EnableFeignClients;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableFeignClients(basePackages = "com.self.growth.integration.test.feign")
public class FeignClientsConfigurationCustom implements RequestInterceptor {

    private final SaTokenContext saTokenContext;

    public FeignClientsConfigurationCustom(SaTokenContext saTokenContext) {
        this.saTokenContext = saTokenContext;
    }

    @Override
    public void apply(RequestTemplate template) {
        final String token = saTokenContext.getToken();
        if (token == null) {
            return;
        }
        template.header(saTokenContext.getKey(), saTokenContext.getToken());
    }
}
```



最后，业务服务的Feign Client如下，就是一个简单的hello请求：

```java
package com.self.growth.integration.test.feign;

import com.self.growth.integration.test.vo.ResResult;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

@FeignClient(
        value = "RecordHelloFeign",
        url = "${test.server.record.url}"
)
public interface RecordHelloClient {

    @RequestMapping(method = RequestMethod.GET, value = "/hello")
    ResResult<String> hello();
}
```



## 测试验证

编写测试类：

下面是测试记录，每次发起请求，都调用下登录服务（登录服务中有做登录后不在进行登录的处理逻辑，即登录一次即可）

```java
package com.self.growth.integration.test;

import com.self.growth.integration.test.service.UserService;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;

public abstract class BaseServerTest {

    @Autowired
    private UserService userService;

    @BeforeEach
    public void login() {
        userService.login();
    }
}
```

访问业务服务的测试：

```java
package com.self.growth.integration.test.record;

import com.self.growth.integration.test.BaseServerTest;
import com.self.growth.integration.test.feign.RecordHelloClient;
import com.self.growth.integration.test.service.UserService;
import com.self.growth.integration.test.vo.ResResult;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@Slf4j
@SpringBootTest
public class RecordServerTest extends BaseServerTest {

    @Autowired
    private RecordHelloClient recordHelloClient;

    @Test
    public void helloTest() {
        ResResult<String> res = recordHelloClient.hello();
        log.info(res.toString());
        Assertions.assertEquals(200, res.getCode());
    }
}
```



结果如下：

```jav
2022-05-29 08:07:10.695  INFO 16348 --- [           main] c.s.g.i.test.service.UserService         : 登录成功
2022-05-29 08:07:11.959  INFO 16348 --- [           main] c.s.g.i.test.record.RecordServerTest     : ResResult(data=hello: 1, code=200, msg=null)
```



## 总结

本文对于Feign共享登录信息进行一次尝试，使用的是定义拦截器加入token到请求头的方式