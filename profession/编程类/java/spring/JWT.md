# JWT
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

import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import io.jsonwebtoken.security.Keys;

import java.security.Key;
import java.util.Date;

/**
 * @author lw1243925457
 */
public class JWT {

    private static final Key key = Keys.secretKeyFor(SignatureAlgorithm.HS256);

    /**
     * 生成 JWT TOKEN
     *
     * @param value
     * @return
     */
    public static String generateRobotToken(String value) {
        return Jwts.builder()
                .setSubject(value)
                .signWith(key)
                .setExpiration(new Date(System.currentTimeMillis() + (60 * 60 * 24 * 7)))
                .compact();
    }

    public static String parseRobotToken(String token) {
        return Jwts.parserBuilder().setSigningKey(key).build().parseClaimsJws(token).getBody().getSubject();
    }
}



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

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.*;

/**
 * @author lw1243925457
 */
@Configuration
public class WebConfig implements WebMvcConfigurer {

    /**
     * 添加拦截器
     */
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        //拦截路径可自行配置多个 可用 ，分隔开
        registry.addInterceptor(new JwtInterceptor()).addPathPatterns("/**");
    }

    /**
     * 跨域支持
     *
     * @param registry
     */
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedHeaders("Access-Control-Allow-Origin",
                        "*",
                        "Access-Control-Allow-Methods",
                        "POST, GET, OPTIONS, PUT, DELETE",
                        "Access-Control-Allow-Headers",
                        "Origin, X-Requested-With, Content-Type, Accept")
                .allowedOrigins("*")
                .allowedMethods("*");
    }
}

```

## 参考链接
- [SpringBoot+JWT实现token验证并将用户信息存储到@注解内](https://segmentfault.com/a/1190000022776284)
- [实战SpringBoot集成JWT实现token验证（附项目地址）](https://blog.csdn.net/weixin_38405253/article/details/107888374)
- [SpringBoot+JWT完成token验证](https://zhuanlan.zhihu.com/p/74345791)
- [SpringBoot系列之前后端接口安全技术JWT](SpringBoot系列之前后端接口安全技术JWT)
- [springboot 集成jwt设置过期时间_传说中的jwt，我们来征服一下](https://blog.csdn.net/weixin_39746282/article/details/111170020?utm_medium=distribute.pc_relevant.none-task-blog-baidujs_title-0&spm=1001.2101.3001.4242)
- [Spring Boot + Security + JWT 实现Token验证+多Provider——登录系统](https://www.cnblogs.com/xxbbtt/p/10982429.html)
- [SpringBoot（六）基于token快速获取用户登录信息](https://blog.csdn.net/lizc_lizc/article/details/99171901)