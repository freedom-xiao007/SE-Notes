# Soul网关源码解析（十）自定义简单插件编写
***
## 简介
&ensp;&ensp;&ensp;&ensp;综合前面所分析的插件处理流程相关知识，此次我们来编写自定义的插件：统计请求在插件链中的经历时长

## 编写准备
&ensp;&ensp;&ensp;&ensp;首先我们先探究一下，一个Plugin是如何加载到上篇文章分析中的 plugins 中的，plugins 代码如下：

&ensp;&ensp;&ensp;&ensp;我们查看下 plugins 的值，发现global也在里面，也就是所有的plugin都是在里面

```java
public class SoulConfiguration {

    @Bean("webHandler")
    public SoulWebHandler soulWebHandler(final ObjectProvider<List<SoulPlugin>> plugins) {
        List<SoulPlugin> pluginList = plugins.getIfAvailable(Collections::emptyList);
        // global plugin 也在里面
        final List<SoulPlugin> soulPlugins = pluginList.stream()
                .sorted(Comparator.comparingInt(SoulPlugin::getOrder)).collect(Collectors.toList());
        soulPlugins.forEach(soulPlugin -> log.info("load plugin:[{}] [{}]", soulPlugin.named(), soulPlugin.getClass().getName()));
        return new SoulWebHandler(soulPlugins);
    }
}
```

&ensp;&ensp;&ensp;&ensp;在上面的调用栈已经中断了，一直翻不到什么有用的东西，换换思路，我们在 globalPlugin 构造函数上打上断点，重启

&ensp;&ensp;&ensp;&ensp;启动后，果然调用栈更新，我们看看调用栈，看到上面的 SoulConfiguration 调用是在 globalPlugin 之前，所以没啥有用的东西

&ensp;&ensp;&ensp;&ensp;我们查看下 globalPlugin 构造函数的调用触发上一节，发现是下面这个： GlobalPluginConfiguration

```java
@Configuration
@ConditionalOnClass(GlobalPlugin.class)
public class GlobalPluginConfiguration {

    @Bean
    public SoulPlugin globalPlugin(final SoulContextBuilder soulContextBuilder) {
        return new GlobalPlugin(soulContextBuilder);
    }
}
```

&ensp;&ensp;&ensp;&ensp;我们仔细看看这个类，它是Spring Configuration，生成bean后注入进去，后面Spring会自己进行操作装配之类

&ensp;&ensp;&ensp;&ensp;我们注意到这个bean返回的是 SoulPlugin ，还记得我们前面文章分析的所有Plugin都是继承于这个的，所以 List<SoulPlugin> 就会自动装配到所有的 plugin。这个细节我也不是很懂，Spring还是不够熟系，后面需要补一补

&ensp;&ensp;&ensp;&ensp;但看到这，我们大致思路就有了：

- 1.写一个自定义插件
- 2.写一个自定义插件的Spring配置，注入进去

## 自定义插件编写
&ensp;&ensp;&ensp;&ensp;首先说明下，插件的编写应该遵循Soul网关的规范，还是应该写到Soul-Plugin这个模块中，但我们只是试验验证，就随意一点，直接写在Soul-Bootstrap中

&ensp;&ensp;&ensp;&ensp;PS：时间有点小紧张，研究规范编写也伤时间，下次一定

### 工程结构
&ensp;&ensp;&ensp;&ensp;此次需要编写两个文件：

- 自定义插件：TimeRecordPlugin
- 自定义插件配置：TimeRecordConfiguration

&ensp;&ensp;&ensp;&ensp;目录结构大致如下：直接在源码的Soul-Bootstrap模块下

```
├─src
│  ├─main
│  │  ├─java
│  │  │  └─org
│  │  │      └─dromara
│  │  │          └─soul
│  │  │              └─bootstrap
│  │  │                  ├─configuration : 放入自定义插件配置
│  │  │                  ├─filter
│  │  │                  └─plugin ：放入自定义插件
│  │  └─resources
```

### 自定义插件编写
&ensp;&ensp;&ensp;&ensp;首先继承 SoulPlugin ，这样能正常注入到datalist中

&ensp;&ensp;&ensp;&ensp;然后编写相应的处理函数，在处理函数中，我们在请求第一次进入到插件的时候，在exchange中放入当前的系统时间

&ensp;&ensp;&ensp;&ensp;模仿 WebClientResponsePlugin ，在plugin链执行返回后，我们取出之前的系统时间，用当前系统时间减去，得到请求在plugin链中的经历时长

&ensp;&ensp;&ensp;&ensp;order方面需要注意， globalPlugin的order为0，通过前面文章的分析，它进行的操作也不小，我们这个插件得在它前面，那我们的order就设置为-1

&ensp;&ensp;&ensp;&ensp;这样，一个自定义插件就写好了，大致代码如下：

```java
import lombok.extern.slf4j.Slf4j;
import org.dromara.soul.plugin.api.SoulPlugin;
import org.dromara.soul.plugin.api.SoulPluginChain;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Slf4j
public class TimeRecordPlugin implements SoulPlugin {

    private final String TIME_RECORD = "time_record";

    @Override
    public Mono<Void> execute(ServerWebExchange exchange, SoulPluginChain chain) {
        exchange.getAttributes().put(TIME_RECORD, System.currentTimeMillis());

        return chain.execute(exchange).then(Mono.defer(() -> {
            Long startTime = exchange.getAttribute(TIME_RECORD);

            if (startTime == null) {
                log.info("Get start time error");
                return Mono.empty();
            }

            long timeRecord = System.currentTimeMillis() - startTime;
            log.info("Plugin time record: " + timeRecord + " ms");
            return Mono.empty();
        }));
    }

    @Override
    public int getOrder() {
        return -1;
    }
}
```

### 自定义插件配置
&ensp;&ensp;&ensp;&ensp;灰常的简单，我们不需要任何东西，所以没有啥参数传入，直接new一个返回即可，代码如下：

```java
import org.dromara.soul.bootstrap.plugin.TimeRecordPlugin;
import org.dromara.soul.plugin.api.SoulPlugin;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConditionalOnClass(TimeRecordPlugin.class)
public class TimeRecordConfiguration {

    @Bean
    public SoulPlugin timeRecordPlugin() {
        return new TimeRecordPlugin();
    }
}
```

### 运行测试
&ensp;&ensp;&ensp;&ensp;我们把Soul-admin、Soul-Bootstrap、Soul-Example-Http给启动起来

&ensp;&ensp;&ensp;&ensp;访问：http://127.0.0.1:9195/http/order/findById?id=1111

&ensp;&ensp;&ensp;&ensp;查看日志，看到明显自定义插件的日志打印，NICE！

```shell script
o.d.s.plugin.httpclient.WebClientPlugin  : The request urlPath is http://192.168.101.104:8188/order/findById?id=1111, retryTimes is 0
o.d.s.bootstrap.plugin.TimeRecordPlugin  : Plugin time record: 9 ms
o.d.soul.plugin.base.AbstractSoulPlugin  : resilience4j selector success match , selector name :http_limiter
o.d.soul.plugin.base.AbstractSoulPlugin  : resilience4j rule success match , rule name :http_limiter
o.d.soul.plugin.base.AbstractSoulPlugin  : divide selector success match , selector name :/http
o.d.soul.plugin.base.AbstractSoulPlugin  : divide rule success match , rule name :/http/order/findById
o.d.s.plugin.httpclient.WebClientPlugin  : The request urlPath is http://192.168.101.104:8188/order/findById?id=1111, retryTimes is 0
o.d.s.bootstrap.plugin.TimeRecordPlugin  : Plugin time record: 10 ms
```

## 总结
&ensp;&ensp;&ensp;&ensp;Soul网关的主要处理分析基本快结束了，下篇写一个总结就准备开始分析另外一个重要的模块：数据同步

&ensp;&ensp;&ensp;&ensp;此篇的自定义插件编写还是比较简单，没有涉及到选择器和规则，但想做类似Divide之类的插件也不是不可以，那就直接把URI判断规则写死

&ensp;&ensp;&ensp;&ensp;因为请求的所有数据都可以获取的，不用到后台只是规则不能动态变化而已

&ensp;&ensp;&ensp;&ensp;有兴趣的老哥可以尝试写一下，还是挺有意思的，哈哈

## Soul网关源码分析文章列表
### Github
- [Soul源码阅读（一） 概览](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB1-%E6%A6%82%E8%A7%88.md)
- [Soul源码阅读（二）代码初步运行](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB2-%E5%88%9D%E6%AD%A5%E8%BF%90%E8%A1%8C.md)
- [Soul源码阅读（三）HTTP请求处理概览](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB3-%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%A6%82%E8%A7%88.md)
- [Soul网关源码阅读（四）Dubbo请求概览](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB4-dubbo%E8%AF%B7%E6%B1%82%E6%A6%82%E8%A7%88.md)
- [Soul网关源码阅读（五）请求类型探索](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB5-%E8%AF%B7%E6%B1%82%E7%B1%BB%E5%9E%8B%E6%8E%A2%E7%B4%A2.md)
- [Soul网关源码阅读（六）Sofa请求处理概览](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB6-sofa%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%A6%82%E8%A7%88.md)
- [Soul网关源码阅读（七）限流插件初探](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB7-%E9%99%90%E6%B5%81%E6%8F%92%E4%BB%B6%E5%88%9D%E6%8E%A2.md)
- [Soul网关源码阅读（八）路由匹配初探](https://github.com/lw1243925457/SE-Notes/blob/0e6931519a84d5c603504b2c6a633698ac793b70/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB8-%E8%B7%AF%E7%94%B1%E5%8C%B9%E9%85%8D%E5%88%9D%E6%8E%A2.md)
- [Soul网关源码阅读（九）插件配置加载初探](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB9-%E6%8F%92%E4%BB%B6%E9%85%8D%E7%BD%AE%E5%8A%A0%E8%BD%BD%E5%88%9D%E6%8E%A2.md)

- [Soul网关源码阅读番外篇（一） HTTP参数请求错误](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%95%AA%E5%A4%96%E7%AF%871-HTTP%E7%A4%BA%E4%BE%8B%E5%8F%82%E6%95%B0%E8%AF%B7%E6%B1%82%E9%94%99%E8%AF%AF.md)

### 掘金
- [Soul网关源码阅读（一） 概览](https://juejin.cn/post/6917864624423436296)
- [Soul网关源码阅读（二）代码初步运行](https://juejin.cn/post/6917865804121767944)
- [Soul网关源码阅读（三）请求处理概览](https://juejin.cn/post/6917866538712334343)
- [Soul网关源码阅读（四）Dubbo请求概览](https://juejin.cn/post/6917867369909977102)
- [Soul网关源码阅读（五）请求类型探索](https://juejin.cn/post/6918575905962983438)
- [Soul网关源码阅读（六）Sofa请求处理概览](https://juejin.cn/post/6918736260467015693)
- [Soul网关源码阅读（七）限流插件初探](https://juejin.cn/post/6919348164944232455/)
- [Soul网关源码阅读（八）路由匹配初探](https://juejin.cn/post/6919774553241550855/)
- [Soul网关源码阅读（九）插件配置加载初探](https://juejin.cn/post/6920074307590684685/)

- [Soul网关源码阅读番外篇（一） HTTP参数请求错误](https://juejin.cn/post/6918947689564471309)