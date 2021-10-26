# Swagger2的使用
***

```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

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

## 参考链接
- [swagger默认访问地址](https://blog.csdn.net/AlbertFly/article/details/80859684)
- [spring boot controller 增加指定前缀的两种方法](https://blog.csdn.net/java_zhangshuai/article/details/107580703)
- [ knife4j](https://doc.xiaominfo.com/knife4j/documentation/get_start.html)