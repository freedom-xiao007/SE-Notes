# Spring Cloud Gateway (八) 定制化filter的配置(路由匹配概览)
***
### 定制化filter的来源
&ensp;&ensp;&ensp;&ensp;非 global filter 从 routeLocator 中获取的，跟踪 routeLocator 看其的来源

```java
# class RoutePredicateHandlerMapping
    protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
        // routeLocator 跟踪堆栈发现是 CachingRouteLocator
		return this.routeLocator.getRoutes()
				.concatMap(route -> Mono.just(route).filterWhen(r -> {
                    ......
				})
				.next()
                    ......
				.map(route -> {
                    ......
				});
	}
```

&ensp;&ensp;&ensp;&ensp;再跟踪 CachingRouteLocator ，发现是从 CompositeRouteLocator 来的

```java
    public CachingRouteLocator(RouteLocator delegate) {
		this.delegate = delegate;

        // 这里提供一种Flux流缓存构造方式，他的意思是，当cache没有的时候，
        // 我们执行this.delegate.getRoutes().sort(AnnotationAwareOrderComparator.INSTANCE)获得Flux流，并把这个结果写入cache这个Map。
        // 这不会马上执行，当有subscribe才会执行。是懒加载方式。
		routes = CacheFlux.lookup(cache, CACHE_KEY, Route.class)
				.onCacheMissResume(this::fetch);
	}

    private Flux<Route> fetch() {
        // 这个方法是一个核心的方法。负责把所有routeDefinition转换为route。是连接RouteDefinitionLocator和RouteLocator的通道
		return this.delegate.getRoutes().sort(AnnotationAwareOrderComparator.INSTANCE);
	}


    @Bean
	@Primary
	@ConditionalOnMissingBean(name = "cachedCompositeRouteLocator")
	// TODO: property to disable composite?
	public RouteLocator cachedCompositeRouteLocator(List<RouteLocator> routeLocators) {
        // routeLocators 里面有大量的 定制化的 filter factory
		return new CachingRouteLocator(new CompositeRouteLocator(Flux.fromIterable(routeLocators)));
	}

    public Flux<Route> getRoutes() {
		return this.delegates.flatMapSequential(RouteLocator::getRoutes);
	}
```

&ensp;&ensp;&ensp;&ensp;跟踪 CompositeRouteLocator 发现是初始化加载就初始化好了，大致流程是知道，细节后面再看，因为这部分属于加载了，后面分析这部分的时候再分析

&ensp;&ensp;&ensp;&ensp;下面回头来看route的匹配，在下面的一段代码中，好像是相关匹配功能的

```java
    protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
		return this.routeLocator.getRoutes()
                // filterWhen 是返回的 true 和 false ，应该是起过滤的作用，但它后面的逻辑没有搞懂，不知道具体是怎么匹配的
				.concatMap(route -> Mono.just(route).filterWhen(r -> {
					exchange.getAttributes().put(GATEWAY_PREDICATE_ROUTE_ATTR, r.getId());
					return r.getPredicate().apply(exchange);
				})
						.doOnError(e -> logger.error(
								"Error applying predicate for route: " + route.getId(),
								e))
						.onErrorResume(e -> Mono.empty()))
				.next()
				.map(route -> {
					if (logger.isDebugEnabled()) {
						logger.debug("Route matched: " + route.getId());
					}
					validateRoute(route, exchange);
					return route;
				});
	}
```

&ensp;&ensp;&ensp;&ensp;跟踪调用栈来到下面这段代码，可以看到返回的bool，但具体是怎么匹配的，没有搞懂

```java
		public Publisher<Boolean> apply(T t) {
			return Mono.from(left.apply(t)).flatMap(
					result -> !result ? Mono.just(false) : Mono.from(right.apply(t)));
		}
```

&ensp;&ensp;&ensp;&ensp;上面到这一直找不到这部分的细节，于是是看了骆老哥的相关于 predicate （断言）的一些学习笔记，知道了断言在一开始需要进行配置加载，接口类是： RoutePredicateFactory ，于是我们搜索定位到这个类，看看有哪些实现类，发现了两个和本次程序示例非常相关的两个： PathRoutePredicateFactory ， MethodRoutePredicateFactory ，寻找匹配相关的代码，给打上断点，在 PathRoutePredicateFactory 中打断点的相关代码如下：

&ensp;&ensp;&ensp;&ensp;这里看到明显的返回布尔值的相关代码，在之前的分析中就是需要返回布尔值，打上断点后，调式确实进入了，到这终于找到了相关的匹配逻辑

```java
# class PathRoutePredicateFactory
	public Predicate<ServerWebExchange> apply(Config config) {
		final ArrayList<PathPattern> pathPatterns = new ArrayList<>();
		synchronized (this.pathPatternParser) {
			pathPatternParser.setMatchOptionalTrailingSeparator(
					config.isMatchOptionalTrailingSeparator());
			config.getPatterns().forEach(pattern -> {
				PathPattern pathPattern = this.pathPatternParser.parse(pattern);
				pathPatterns.add(pathPattern);
			});
		}
		return new GatewayPredicate() {
			@Override
			public boolean test(ServerWebExchange exchange) {
				PathContainer path = parsePath(
						exchange.getRequest().getURI().getRawPath());

				Optional<PathPattern> optionalPathPattern = pathPatterns.stream()
						.filter(pattern -> pattern.matches(path)).findFirst();

				if (optionalPathPattern.isPresent()) {
					PathPattern pathPattern = optionalPathPattern.get();
					traceMatch("Pattern", pathPattern.getPatternString(), path, true);
					PathMatchInfo pathMatchInfo = pathPattern.matchAndExtract(path);
					putUriTemplateVariables(exchange, pathMatchInfo.getUriVariables());
					return true;
				}
				else {
					traceMatch("Pattern", config.getPatterns(), path, false);
					return false;
				}
			}

			@Override
			public String toString() {
				return String.format("Paths: %s, match trailing slash: %b",
						config.getPatterns(), config.isMatchOptionalTrailingSeparator());
			}
		};
	}
```

&ensp;&ensp;&ensp;&ensp;Path 的匹配逻辑可能有点复杂，看起来不是很直观，但 Method 的就很清晰了，大致如下,明显的匹配逻辑：

```java
# class MethodRoutePredicateFactory
	public Predicate<ServerWebExchange> apply(Config config) {
		return new GatewayPredicate() {
			@Override
			public boolean test(ServerWebExchange exchange) {
				HttpMethod requestMethod = exchange.getRequest().getMethod();
				return stream(config.getMethods())
						.anyMatch(httpMethod -> httpMethod == requestMethod);
			}
		};
	}
```

&ensp;&ensp;&ensp;&ensp;接下来看看匹配的和或非之类的逻辑是如何实现的，相关的代码都在之前阻塞的 AsyncPredicate ，配置组合是在 combinePredicates ,可以大致看出在程序启动的时候就配置加载好了。和或非的匹配具体逻辑大致应该是递归之类实现的吧

```java
	private AsyncPredicate<ServerWebExchange> combinePredicates(
			RouteDefinition routeDefinition) {
		List<PredicateDefinition> predicates = routeDefinition.getPredicates();
		if (predicates == null || predicates.isEmpty()) {
			// this is a very rare case, but possible, just match all
			return AsyncPredicate.from(exchange -> true);
		}
		AsyncPredicate<ServerWebExchange> predicate = lookup(routeDefinition,
				predicates.get(0));

		for (PredicateDefinition andPredicate : predicates.subList(1,
				predicates.size())) {
			AsyncPredicate<ServerWebExchange> found = lookup(routeDefinition,
					andPredicate);
			predicate = predicate.and(found);
		}

		return predicate;
	}
```