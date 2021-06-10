# Spring Cloud Gateway （三） Filter 链的生成与触发
***
## 简介
&ensp;&ensp;&ensp;&ensp;在上篇中分析了请求在Filter中的传播过程，但不同的请求，应该是有不同的各自的定制的filter，今天来看看如何对请求定制filter链

## 分析过程
&ensp;&ensp;&ensp;&ensp;我们找到上次filter链的循环部分，里面对filter链进行了循环遍历，逐个对每个filter进行了调用，在filter里面对全局的exchange进行了修改，exchange承担了整个流程中的数据。

&ensp;&ensp;&ensp;&ensp;在 FilteringWebHandler 中找到了如何构造特定请求的定制filter链的代码。在下面的代码中，特定请求的filter存放到了exchange中，和全局的filter放到一起排序，形成了上次的filter链路

```java
public class FilteringWebHandler implements WebHandler {
	@Override
	public Mono<Void> handle(ServerWebExchange exchange) {
		// 这里是获取特定转发的filter列表
		Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
		List<GatewayFilter> gatewayFilters = route.getFilters();

		// 获取全局filter列表，和上面的排序组合成新的filter
		List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
		combined.addAll(gatewayFilters);
		// TODO: needed or cached?
		AnnotationAwareOrderComparator.sort(combined);

		if (logger.isDebugEnabled()) {
			logger.debug("Sorted gatewayFilterFactories: " + combined);
		}

		// 触发调用
		return new DefaultGatewayFilterChain(combined).filter(exchange);
	}
}
```

&ensp;&ensp;&ensp;&ensp;想查询定位到exchange的路由路径独有filter是如何来的，但没有思路，后面看到Get Route的下面这句：

```java
Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
```

&ensp;&ensp;&ensp;&ensp;查看相关代码中如何设置的代码，假设在代码中存在下面这句，把route放入其中的语句：

```java
exchange.getAttributes().put(GATEWAY_ROUTE_ATTR
```

&ensp;&ensp;&ensp;&ensp;经过搜索，找到了下面的类：RoutePredicateHandlerMapping

```java
public class RoutePredicateHandlerMapping extends AbstractHandlerMapping {

	@Override
	protected Mono<?> getHandlerInternal(ServerWebExchange exchange) {
		// don't handle requests on management port if set and different than server port
		if (this.managementPortType == DIFFERENT && this.managementPort != null
				&& exchange.getRequest().getURI().getPort() == this.managementPort) {
			return Mono.empty();
		}
		exchange.getAttributes().put(GATEWAY_HANDLER_MAPPER_ATTR, getSimpleName());

		return lookupRoute(exchange)
				// .log("route-predicate-handler-mapping", Level.FINER) //name this
				.flatMap((Function<Route, Mono<?>>) r -> {
					exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
					if (logger.isDebugEnabled()) {
						logger.debug(
								"Mapping [" + getExchangeDesc(exchange) + "] to " + r);
					}

					// r 是它独有的filter 处理器
					exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, r);
					// webHandler 是全局链？
					return Mono.just(webHandler);
				}).switchIfEmpty(Mono.empty().then(Mono.fromRunnable(() -> {
					exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
					if (logger.isTraceEnabled()) {
						logger.trace("No RouteDefinition found for ["
								+ getExchangeDesc(exchange) + "]");
					}
				})));
	}
}
```

&ensp;&ensp;&ensp;&ensp;在上面的代码中，指定路由路径的定制filter是从这传入的，并且debug可以看到，这个可以跳转到filter的开始类中，那这个类是承接了route和filter的最后一个route相关的类。

&ensp;&ensp;&ensp;&ensp;但其中还要一些疑问，为啥要把webHandler放入Mono中，不应该是系统启动的时候就配置好的吗，这里和我想象的不一样，后序看看能不能捋一捋