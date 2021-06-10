# Spring Cloud Gateway (六) 自定义 Global Filter
***
## 简介
&ensp;&ensp;&ensp;&ensp;在前面五篇的分析中，对 Spring Cloud Gateway 的 filter 组件有了一个大概的认知，今天就练练手，写一个统计请求返回时长的 global filter

## 思路整理
&ensp;&ensp;&ensp;&ensp;阅读官方文档和在前面读源码的过程中，大致知道 global filter 需要继承 GlobalFilter 、 Ordered

- GlobalFilter : 需要重写 filter 方法，在其中实现自己的处理逻辑，很大部分都是对 exchange 进行操作，取值赋值等等
- Ordered ： 需要重写 getOrder 方法，决定自定义 filter 在链中的位置，按照数字大小进行的排序

### 自定义时长统计 filter 编写
&ensp;&ensp;&ensp;&ensp;这里就简单的借鉴下 GatewayMetricsFilter 的写法，在 filter 链触发完成后，无论失败或者成功，都进行统计。filter 次序就简单的取 WRITE_RESPONSE_FILTER_ORDER 之后（用它定义的次序减一）。代码大致如下：


```java
/**
 * 统计一次请求在filter链中的时长
 *
 * @author lw1243925457
 */
public class DurationStatisticsFilter implements GlobalFilter, Ordered {

	private static final Log log = LogFactory.getLog(DurationStatisticsFilter.class);
	private static final String START_STAMP = "startStamp";

	/**
	 * 请求响应时长统计
	 * 1.首先将请求进来时的时间戳写入到 exchange中
	 * 2.filter链走完后，无论成功或者失败，都视为完成，从exchange取出开始时间戳，打印时长信息
	 * @param exchange the current server exchange
	 * @param chain provides a way to delegate to the next filter
	 * @return mono
	 */
	@Override
	public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
		log.debug("duration statistics filter");

		exchange.getAttributes().put(START_STAMP, System.currentTimeMillis());
		return chain.filter(exchange)
				.doFinally(f -> printDurationTime(exchange));
	}

	private void printDurationTime(ServerWebExchange exchange) {
		long startStamp = exchange.getAttribute(START_STAMP);
		long endStamp = System.currentTimeMillis();
		log.debug("duration filter time : " + (endStamp - startStamp) + " ms");
	}

	/**
	 * 这里简单位于 NettyWriteResponseFilter 之后吧
	 * @return order
	 */
	@Override
	public int getOrder() {
		return WRITE_RESPONSE_FILTER_ORDER - 1;
	}
}
```

### 主函数配置运行
&ensp;&ensp;&ensp;&ensp;需要在主函数中配置 bean，整个代码大致如下：

```java
@SpringBootApplication
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}

	@Bean
	public RouteLocator myRoutes(RouteLocatorBuilder builder) {
		return builder.routes()
				.route(p -> p
						.path("/")
						.filters(f -> f
								.addRequestParameter("test", "test")
								.addResponseHeader("return", "return")
								.retry(retryConsumer)
						)
						.uri("http://localhost:8082/"))
				.build();
	}

	@Bean
	public GlobalFilter durationStatisticsFilter() {
		return new DurationStatisticsFilter();
	}
}
```

&ensp;&ensp;&ensp;&ensp;很简单，大致就这些，运行起来，浏览器访问，可以看到下面的输出，over

```shell
o.s.c.g.sample.DurationStatisticsFilter  : duration statistics filter
o.s.c.g.sample.DurationStatisticsFilter  : duration filter time : 349 ms
```

## Filter 相关分析记录
- [Spring cloud Gateway (一) 代码拉取与运行示例](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/java/spring/springcloudGateway/SpringCloudGateway%E6%A6%82%E8%A7%88.md)
- [Spring cloud Gateway（二） 一个Http请求的流程解析](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/java/spring/springcloudGateway/%E6%B5%81%E7%A8%8B%E7%B1%BB.md)
- [Spring Cloud Gateway （三） Filter 链](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/java/spring/springcloudGateway/Filter%E9%93%BE.md)
- [Spring Cloud Gateway(四) 请求重试机制](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/java/spring/springcloudGateway/%E9%87%8D%E8%AF%95%E6%9C%BA%E5%88%B6.md)
- [Spring Cloud Gateway(五) 限流](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/java/spring/springcloudGateway/%E9%99%90%E6%B5%81.md)
- [Spring Cloud Gateway (六) 自定义 Global Filter](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/java/spring/springcloudGateway/%E8%87%AA%E5%AE%9A%E4%B9%89GlobalFilter.md)