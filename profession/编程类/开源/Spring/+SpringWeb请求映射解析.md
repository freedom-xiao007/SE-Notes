# Spring 源码解析 -- SpringWeb请求映射解析
***
## 简介
基于上篇请求路径初步探索，了解到了一个请求到具体处理方法的大致路径，本篇就继续探索，看下路径是如何匹配到处理方法的

## 概览
基于上篇：[Spring Web 请求初探](https://juejin.cn/post/6980529362969821192/)

我们大致定位到了请求路径到处理方法的关键代码在类中：DispatcherServlet.java

具体的代码如下：

```java
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

mappedHandler 的获取就是关键了，下面我们就具体看下如何获取请求的mappedHandler的

## 源码解析
### 获取 mappedHandler
在类：DispatcherServlet.java 中，我们定位到 mappedHandler 获取的关键代码

```java
// Determine handler for the current request.
mappedHandler = getHandler(processedRequest);
if (mappedHandler == null) {
    noHandlerFound(processedRequest, response);
    return;
}
```

通过上面的代码，我们可以看到是通过请求进行获取，我们还注意到了为null的情况，如下代码，大意就是经典的404

```java
/**
* No handler found -> set appropriate HTTP response status.
* @param request current HTTP request
* @param response current HTTP response
* @throws Exception if preparing the response failed
*/
protected void noHandlerFound(HttpServletRequest request, HttpServletResponse response) throws Exception {
	if (pageNotFoundLogger.isWarnEnabled()) {
		pageNotFoundLogger.warn("No mapping for " + request.getMethod() + " " + getRequestUri(request));
	}
	if (this.throwExceptionIfNoHandlerFound) {
		throw new NoHandlerFoundException(request.getMethod(), getRequestUri(request),
				new ServletServerHttpRequest(request).getHeaders());
	}
	else {
		response.sendError(HttpServletResponse.SC_NOT_FOUND);
	}
}
```

我们进入 getHandler 函数，看看其具体处理逻辑，代码如下：

```java
/**
* Return the HandlerExecutionChain for this request.
* <p>Tries all handler mappings in order.
* @param request current HTTP request
* @return the HandlerExecutionChain, or {@code null} if no handler could be found
*/
@Nullable
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
	for (HandlerMapping mapping : this.handlerMappings) {
		HandlerExecutionChain handler = mapping.getHandler(request);
		if (handler != null) {
			return handler;
		}
	}
    }
	return null;
}
```

从上面代码大意可以看出：循环遍历一个关于请求匹配的Map，如果某个Handler匹配成功则返回

这个Map是如何初始化、具体包含哪些内容？

HandlerExecutionChain 具体是什么？

这部分疑问后面再看，感觉是个大工程，当前并不影响本篇的解析

### 请求与Mapper匹配
接着跟下去，到类：AbstractHandlerMethodMapping.java

我们来到了一个比较关键的处理，我们可以看到这里有相关的匹配逻辑，里面有很多的处理，有些目前都不知道有啥作用，后面查查资料后咱们再回过头来看看

```java
	@Override
	@Nullable
	public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		Object handler = getHandlerInternal(request);

		// 什么情况下会匹配到默认的Handler？
		if (handler == null) {
			handler = getDefaultHandler();
		}
		if (handler == null) {
			return null;
		}

		// 什么情况下获取到的Bean是一个String？
		// 需要从 obtainApplicationContext 中获取
		// Bean name or resolved handler?
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = obtainApplicationContext().getBean(handlerName);
		}

		// 什么情况下会用到缓存？
		// Ensure presence of cached lookupPath for interceptors and others
		if (!ServletRequestPathUtils.hasCachedPath(request)) {
			initLookupPath(request);
		}

		HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

		if (logger.isTraceEnabled()) {
			logger.trace("Mapped to " + handler);
		}
		else if (logger.isDebugEnabled() && !DispatcherType.ASYNC.equals(request.getDispatcherType())) {
			logger.debug("Mapped to " + executionChain.getHandler());
		}

		if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
			CorsConfiguration config = getCorsConfiguration(handler, request);
			if (getCorsConfigurationSource() != null) {
				CorsConfiguration globalConfig = getCorsConfigurationSource().getCorsConfiguration(request);
				config = (globalConfig != null ? globalConfig.combine(config) : config);
			}
			if (config != null) {
				config.validateAllowCredentials();
			}
			executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
		}

		return executionChain;
	}
```

其中的 getHandlerExecutionChain 还挺可疑，感觉也是关键代码，给设置了一堆的我们熟悉的 Interceptors,这里做个标记，后面再探索下

```java
	protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
		HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
				(HandlerExecutionChain) handler : new HandlerExecutionChain(handler));

		for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
			if (interceptor instanceof MappedInterceptor) {
				MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
				if (mappedInterceptor.matches(request)) {
					chain.addInterceptor(mappedInterceptor.getInterceptor());
				}
			}
			else {
				chain.addInterceptor(interceptor);
			}
		}
		return chain;
	}
```

### Handler匹配
下面就跟着断点来到了：AbstractHandlerMapping.java

看到下面的关键代码：在查找匹配的时候上了一个锁，进入具体的查找逻辑

```java
@Override
@Nullable
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
	// 这里一些处理得到我们的请求路径：”/"
	String lookupPath = initLookupPath(request);
	this.mappingRegistry.acquireReadLock();
	try {
		HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
		return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
	}
	finally {
		this.mappingRegistry.releaseReadLock();
	}
}
```

我们进入类：AbstractHandlerMethodMapping.java 中，查看具体的处理逻辑

我们可以看到这里进行了一系列的匹配，如果为空还能取默认值，这个默认值是个啥？

并且还有多个匹配上的还不会报错的情况，瞬间感觉Web这块还是有很多门道啊，但这些细节后面我们再看看，继续看下匹配的逻辑

```java
	@Nullable
	protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
		List<Match> matches = new ArrayList<>();
		// 在这里获取到了相关的Handler，就是一个简单的Map取值（如后面的代码），那个Map如何初始化的得后面看看了
		List<T> directPathMatches = this.mappingRegistry.getMappingsByDirectPath(lookupPath);
		if (directPathMatches != null) {
			addMatchingMappings(directPathMatches, matches, request);
		}

		// 如果没有匹配上，这个默认值又是如何处理的呢？
		if (matches.isEmpty()) {
			addMatchingMappings(this.mappingRegistry.getRegistrations().keySet(), matches, request);
		}

		// 这里看到一些非常有趣的处理
		// 如果同时匹配上了两个，好像不是一定报错？
		if (!matches.isEmpty()) {
			Match bestMatch = matches.get(0);
			if (matches.size() > 1) {
				Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
				matches.sort(comparator);
				bestMatch = matches.get(0);
				if (logger.isTraceEnabled()) {
					logger.trace(matches.size() + " matching mappings: " + matches);
				}
				// 这个的返回是什么意思？
				if (CorsUtils.isPreFlightRequest(request)) {
					for (Match match : matches) {
						if (match.hasCorsConfig()) {
							return PREFLIGHT_AMBIGUOUS_MATCH;
						}
					}
				}
				else {
					Match secondBestMatch = matches.get(1);
					if (comparator.compare(bestMatch, secondBestMatch) == 0) {
						Method m1 = bestMatch.getHandlerMethod().getMethod();
						Method m2 = secondBestMatch.getHandlerMethod().getMethod();
						String uri = request.getRequestURI();
						throw new IllegalStateException(
								"Ambiguous handler methods mapped for '" + uri + "': {" + m1 + ", " + m2 + "}");
					}
				}
			}
			request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.getHandlerMethod());
			handleMatch(bestMatch.mapping, lookupPath, request);
			return bestMatch.getHandlerMethod();
		}
		else {
			return handleNoMatch(this.mappingRegistry.getRegistrations().keySet(), lookupPath, request);
		}
	}
```

一个简单的Map取值（如后面的代码），那个Map如何初始化的得后面看看了

```java
class MappingRegistry {

	private final MultiValueMap<String, T> pathLookup = new LinkedMultiValueMap<>();

	@Nullable
	public List<T> getMappingsByDirectPath(String urlPath) {
		return this.pathLookup.get(urlPath);
	}
}
```

到这里就匹配成功并返回了，但通过调试看到不是我们熟悉的类和方法

继续看看这行的具体处理逻辑：return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);

看看类 HandlerMethod 的一些方法和构造函数：

下面函数我们看到了属性的获取Bean

```java
	public HandlerMethod createWithResolvedBean() {
		Object handler = this.bean;
		if (this.bean instanceof String) {
			Assert.state(this.beanFactory != null, "Cannot resolve bean name without BeanFactory");
			String beanName = (String) this.bean;
			handler = this.beanFactory.getBean(beanName);
		}
		return new HandlerMethod(this, handler);
	}
```

在其构造函数中，直接将Bean和Method进行了初始化，这个如何初始化的我们后面再看

```java
	public HandlerMethod(Object bean, Method method) {
		Assert.notNull(bean, "Bean is required");
		Assert.notNull(method, "Method is required");
		this.bean = bean;
		this.beanFactory = null;
		this.beanType = ClassUtils.getUserClass(bean);
		this.method = method;
		this.bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
		this.parameters = initMethodParameters();
		evaluateResponseStatus();
		this.description = initDescription(this.beanType, this.method);
	}
```

## 总结
本篇文章探索了下具体的请求如何匹配到处理方法的相关处理函数,了解到了核心逻辑就是遍历一个： handlerMappings

```java
@Nullable
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
	for (HandlerMapping mapping : this.handlerMappings) {
		HandlerExecutionChain handler = mapping.getHandler(request);
		if (handler != null) {
			return handler;
		}
	}
    }
	return null;
}
```

后面的具体匹配逻辑也有很多值得探索的细节，但接下来更重要的是弄清楚这个: handlerMappings ,是如何初始化和加载的

下面的文章我们会继续探索