# Swagger2的使用
***

这是我参与2022首次更文挑战的第15天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7052884569032392740)

## 简介
在日常开发中，自动API接口文档能大大简化我们的操作，本篇将介绍Swagger的常规用法和可以按照版本进行分组的配置用法

## Swagger使用
### maven依赖
在新版本中，直接使用knife4j即可，非常的好用，依赖如下：

```pom
 <dependency>
            <groupId>com.github.xiaoymin</groupId>
            <artifactId>knife4j-spring-boot-starter</artifactId>
            <!--在引用时请在maven中央仓库搜索最新版本号-->
            <version>2.0.2</version>
        </dependency>
```

### 常规用法
通过下面的简单配置即可使用，在Controller方法中加上注解即可，生成全量的API文档

有个缺点是没有版本划分，如果需要做接口测试，没有办法快速定位当前版本新发布了哪些接口和修改了哪些接口

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.info.BuildProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

/**
 * Configuration class for Swagger API document.
 **/
@Slf4j
@Configuration
@EnableSwagger2
public class SwaggerConfiguration {
    
    @Value("${rpa.swagger.enable:true}")
    private boolean enable;
    
    private final BuildProperties buildProperties;
    
    public SwaggerConfiguration(@Autowired(required = false) final BuildProperties buildProperties) {
        this.buildProperties = buildProperties;
    }
    
    /**
     * Configure The Docket with Swagger.
     *
     * @return Docket {@linkplain Docket}
     */
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(this.apiInfo())
                .enable(enable)
                .select()
                .apis(RequestHandlerSelectors.basePackage(""))
                .paths(PathSelectors.any())
                .build();
    }
    
    /**
     * Fetch version information from pom.xml and set title, version, description,
     * contact for Swagger API document.
     *
     * @return Api info
     */
    private ApiInfo apiInfo() {
        String version = "1.0.0";
        if (buildProperties != null) {
            version = buildProperties.getVersion();
        }
        log.info("http://localhost:8080/swagger-ui.html");
        return new ApiInfoBuilder()
                .title("API")
                .description("")
                .version(version)
                .build();
    }
}
```

### 能按照版本进行筛选
首先自定义一个版本注解，开发新版本的时候，增加对应的版本即可

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ApiVersion {

    Version[] value();

    enum Version {
        MASTER("master"),
        V3_3_0128("V3.3.0128")
        ;

        private String display;

        Version(String display) {
            this.display = display;
        }

        public String getDisplay() {
            return display;
        }
    }
}
```

然后定义Swagger配置：

```java
import com.google.common.base.Function;
import com.google.common.base.Joiner;
import com.google.common.base.Optional;
import com.google.common.base.Predicate;
import com.google.common.collect.FluentIterable;
import com.google.common.collect.Iterables;
import com.google.common.collect.Multimap;
import com.google.common.collect.Multimaps;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Primary;
import org.springframework.core.annotation.AnnotationAwareOrderComparator;
import org.springframework.stereotype.Component;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.service.*;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spi.service.DocumentationPlugin;
import springfox.documentation.spi.service.contexts.SecurityContext;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.spring.web.plugins.DocumentationPluginsManager;
import springfox.documentation.swagger2.annotations.EnableSwagger2;
import swagger.ApiVersion;
import swagger.SwaggerPluginRegistry;

import java.util.*;

@Slf4j
@Component
@Primary
@ConditionalOnProperty(prefix = "swagger", value = {"enable"}, havingValue = "true")
@EnableSwagger2
public class SwaggerDocumentationPluginsManager extends DocumentationPluginsManager {

    @Override
    public Collection<DocumentationPlugin> documentationPlugins() throws IllegalStateException {
        List<DocumentationPlugin> plugins = registry().getPlugins();
        ensureNoDuplicateGroups(plugins);
        if (plugins.isEmpty()) {
            return new ArrayList(Collections.singleton(defaultDocumentationPlugin()));
        }
        return plugins;
    }

    private void ensureNoDuplicateGroups(List<DocumentationPlugin> allPlugins) throws IllegalStateException {
        Multimap<String, DocumentationPlugin> plugins = Multimaps.index(allPlugins, byGroupName());
        Iterable<String> duplicateGroups = FluentIterable.from(plugins.asMap().entrySet()).filter(duplicates()).transform(toGroupNames());
        if (Iterables.size(duplicateGroups) > 0) {
            throw new IllegalStateException(String.format("Multiple Dockets with the same group name are not supported. "
                    + "The following duplicate groups were discovered. %s", Joiner.on(',').join(duplicateGroups)));
        }
    }

    private Function<? super DocumentationPlugin, String> byGroupName() {
        return (Function<DocumentationPlugin, String>) input -> Optional.fromNullable(input.getGroupName()).or("default");
    }

    private Function<? super Map.Entry<String, Collection<DocumentationPlugin>>, String> toGroupNames() {
        return (Function<Map.Entry<String, Collection<DocumentationPlugin>>, String>) input -> input.getKey();
    }

    private static Predicate<? super Map.Entry<String, Collection<DocumentationPlugin>>> duplicates() {
        return (Predicate<Map.Entry<String, Collection<DocumentationPlugin>>>) input -> input.getValue().size() > 1;
    }

    private DocumentationPlugin defaultDocumentationPlugin() {
        return new Docket(DocumentationType.SWAGGER_2);
    }

    private SwaggerPluginRegistry registry() {
        List<Docket> list = new ArrayList<>();
        for (ApiVersion.Version version : ApiVersion.Version.values()) {
            Docket docket = new Docket(DocumentationType.SWAGGER_2)
                    .apiInfo(apiInfo())
                    .groupName(version.getDisplay())
                    .select()
                    .apis(input -> {
                        log.info(input.toString());
                        if (ApiVersion.Version.MASTER.equals(version)) {
                            return true;
                        }
                        ApiVersion apiVersion = input.getHandlerMethod().getMethodAnnotation(ApiVersion.class);
                        if (apiVersion != null && Arrays.asList(apiVersion.value()).contains(version)) {
                            return true;
                        }
                        return false;
                    })
                    .paths(PathSelectors.any())
                    .build()
                    .securityContexts(securityContexts());

            list.add(docket);
        }

        return new SwaggerPluginRegistry(list, new AnnotationAwareOrderComparator());
    }

    private ApiInfo apiInfo() {
        Contact contact = new Contact("技术部", "", "");
        return new ApiInfoBuilder()
                .title("ofcoder接口文档")
                .description("ofcoder接口文档")
                .version("1.0.0")
                .contact(contact)
                .build();
    }

    private List<SecurityContext> securityContexts() {
        List<SecurityContext> arrayList = new ArrayList<>();
        arrayList.add(SecurityContext.builder()
                .securityReferences(defaultAuth())
                .build());
        return arrayList;
    }

    private List<SecurityReference> defaultAuth() {
        AuthorizationScope authorizationScope = new AuthorizationScope("global", "accessEverything");
        AuthorizationScope[] authorizationScopes = new AuthorizationScope[1];
        authorizationScopes[0] = authorizationScope;
        List<SecurityReference> arrayList = new ArrayList<>();
        arrayList.add(new SecurityReference("Authorization", authorizationScopes));
        return arrayList;
    }
}
```

使用示例如下：

```java
@Api(tags = "接口")
@RequestMapping("/")
@RestController
@AllArgsConstructor
public class Controller {

    @ApiVersion({ApiVersion.Version.V3_3_0128})
    @ApiOperation(value = "删除")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "id", value = "ID"),
    })
    @DeleteMapping("/delete/{id}")
    public RespResult<String> delete(@PathVariable(value = "id") long id) {
        return RespResult.buildSuccessResult("success");
    }
}
```

## 文档访问地址
在线接口文档：http://{ip}:{port}/doc.html#/home, 例如：http://127.0.0.1:8000/doc.html#/home


## 参考链接
- [swagger默认访问地址](https://blog.csdn.net/AlbertFly/article/details/80859684)
- [spring boot controller 增加指定前缀的两种方法](https://blog.csdn.net/java_zhangshuai/article/details/107580703)
- [ knife4j](https://doc.xiaominfo.com/knife4j/documentation/get_start.html)