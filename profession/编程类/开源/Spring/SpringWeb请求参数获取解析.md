# Spring源码解析 -- SpringWeb请求参数获取解析
***
## 简介
在文章：[Spring Web 请求初探](https://juejin.cn/post/6980529362969821192/)中，我们看到最后方法反射调用的相关代码，本篇文章就探索其中的参数是如何从请求中获取的

## 概览
方法反射调用的关键代码如下：

```java
public class InvocableHandlerMethod {
	protected Object doInvoke(Object... args) throws Exception {
		Method method = getBridgedMethod();
		ReflectionUtils.makeAccessible(method);
		try {
			if (KotlinDetector.isSuspendingFunction(method)) {
				return CoroutinesUtils.invokeSuspendingFunction(method, getBean(), args);
			}
			return method.invoke(getBean(), args);
		}
		catch (IllegalArgumentException ex) {
			......
		}
	}
}
```

现在我们更新我们的代码如下，增加接个参数：

```java
import com.example.springexample.vo.User;
import org.springframework.web.bind.annotation.*;

@RestController
public class HelloWorld {

    @GetMapping("/")
    public String helloWorld(@RequestParam(value = "id") Integer id,
                             @RequestParam(value = "name") String name) {
        return "Hello world:" + id;
    }

    @GetMapping("/test1")
    public String helloWorld1(@RequestHeader(value = "id") Integer id) {
        return "Hello world:" + id;
    }

    @PostMapping("/test2")
    public String helloWorld2(@RequestBody User user) {
        return "Hello world:" + user.toString();
    }
}
```

发起下面的请求,在上面反射调用的代码中打上断点，跟踪调用栈分析：

```sh
GET http://localhost:8080/?id=1&id=2

POST http://localhost:8080/test2
Content-Type: application/json

{"name": "name", "password": "password"}
```

## 源码解析
首先查看调用栈，我们找到了参数最开始获取的相关代码，在下面的代码中获取相关的参数：

```java
	# InvocableHandlerMethod.java
	@Nullable
	public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
		// 得到了相关的参数
		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
		if (logger.isTraceEnabled()) {
			logger.trace("Arguments: " + Arrays.toString(args));
		}
		return doInvoke(args);
	}
```

继续跟踪看看函数：getMethodArgumentValues

```java
	# InvocableHandlerMethod.java
	protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
		// 这里获得了参数信息列表，比如类型之类的
		MethodParameter[] parameters = getMethodParameters();
		if (ObjectUtils.isEmpty(parameters)) {
			return EMPTY_ARGS;
		}

		Object[] args = new Object[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			MethodParameter parameter = parameters[i];
			parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
			// 这里进行了一次参数查找，但没有匹配上
			args[i] = findProvidedArgument(parameter, providedArgs);
			if (args[i] != null) {
				continue;
			}
			if (!this.resolvers.supportsParameter(parameter)) {
				throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
			}
			try {
				// 最终在这里匹配上了参数
				args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
			}
			catch (Exception ex) {
				// Leave stack trace for later, exception may actually be resolved and handled...
				if (logger.isDebugEnabled()) {
					String exMsg = ex.getMessage();
					if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
						logger.debug(formatArgumentError(parameter, exMsg));
					}
				}
				throw ex;
			}
		}
		return args;
	}
```

在上面的代码中，我们可以看到大致下面的逻辑：

- 获取方法调用所需的参数信息
- 尝试获取具体参数值：findProvidedArgument
- 尝试获取具体参数值：this.resolvers.resolveArgument
- 抛出未获取异常

findProvidedArgument 没怎么看懂，日后在看下

我们先跟着函数：this.resolvers.resolveArgument

```java
	# HandlerMethodArgumentResolverComposite.java
	@Override
	@Nullable
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
		// 获取参数解析器
		HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
		if (resolver == null) {
			throw new IllegalArgumentException("Unsupported parameter type [" +
					parameter.getParameterType().getName() + "]. supportsParameter should be called first.");
		}
		// 返回解析结果
		return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
	}

	@Nullable
	private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
		// 首先查找缓存，如果缓存不存在则遍历查找参数解析器列表
		HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
		if (result == null) {
			for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {
				if (resolver.supportsParameter(parameter)) {
					result = resolver;
					this.argumentResolverCache.put(parameter, result);
					break;
				}
			}
		}
		return result;
	}
```

在上面的函数中，首先取获取对应参数的解析器，关键函数就是： supportsParameter

解析器目前调试发现有32个，本地请求：GET http://localhost:8080?id=1&id=2 匹配上的是： RequestParamMethodArgumentResolver.java ，其判断的代码如下：

```java
	# RequestParamMethodArgumentResolver.java
	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		if (parameter.hasParameterAnnotation(RequestParam.class)) {
			if (Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
				RequestParam requestParam = parameter.getParameterAnnotation(RequestParam.class);
				return (requestParam != null && StringUtils.hasText(requestParam.name()));
			}
			else {
				return true;
			}
		}
		else {
			if (parameter.hasParameterAnnotation(RequestPart.class)) {
				return false;
			}
			parameter = parameter.nestedIfOptional();
			if (MultipartResolutionDelegate.isMultipartArgument(parameter)) {
				return true;
			}
			else if (this.useDefaultResolution) {
				return BeanUtils.isSimpleProperty(parameter.getNestedParameterType());
			}
			else {
				return false;
			}
		}
	}
```

大致就是看看是否有相关的注解、符合要求之类的，这部分就不细看

我们继续看看具体参数获取处理：

```java
	# AbstractNamedValueMethodArgumentResolver.java
	@Override
	@Nullable
	public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
		// 获取参数名称，GET http://localhost:8080?id=1&id=2 就是 id
		NamedValueInfo namedValueInfo = getNamedValueInfo(parameter);
		// 方法调用的信息，如类、参数类型之类的
		MethodParameter nestedParameter = parameter.nestedIfOptional();

		// 看着想是参数效验？没完全看懂，先跳过
		Object resolvedName = resolveEmbeddedValuesAndExpressions(namedValueInfo.name);
		if (resolvedName == null) {
			throw new IllegalArgumentException(
					"Specified name must not resolve to null: [" + namedValueInfo.name + "]");
		}
		// 获取参数值
		Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest);
		if (arg == null) {
			if (namedValueInfo.defaultValue != null) {
				arg = resolveEmbeddedValuesAndExpressions(namedValueInfo.defaultValue);
			}
			else if (namedValueInfo.required && !nestedParameter.isOptional()) {
				handleMissingValue(namedValueInfo.name, nestedParameter, webRequest);
			}
			arg = handleNullValue(namedValueInfo.name, arg, nestedParameter.getNestedParameterType());
		}
		else if ("".equals(arg) && namedValueInfo.defaultValue != null) {
			arg = resolveEmbeddedValuesAndExpressions(namedValueInfo.defaultValue);
		}
		// 在下面的函数中会进行参数转换
		// 比如在请求：GET http://localhost:8080?id=1&id=2
		// 得到的结果是一个数组，有两个值，但转换后直接就得到了 1
		// 这部分的细节日后回过头再来看
		if (binderFactory != null) {
			WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
			try {
				arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter);
			}
			catch (ConversionNotSupportedException ex) {
				throw new MethodArgumentConversionNotSupportedException(arg, ex.getRequiredType(),
						namedValueInfo.name, parameter, ex.getCause());
			}
			catch (TypeMismatchException ex) {
				throw new MethodArgumentTypeMismatchException(arg, ex.getRequiredType(),
						namedValueInfo.name, parameter, ex.getCause());
			}
			// Check for null value after conversion of incoming argument value
			if (arg == null && namedValueInfo.defaultValue == null &&
					namedValueInfo.required && !nestedParameter.isOptional()) {
				handleMissingValueAfterConversion(namedValueInfo.name, nestedParameter, webRequest);
			}
		}

		handleResolvedValue(arg, namedValueInfo.name, parameter, mavContainer, webRequest);

		return arg;
	}
```

上面的函数是一处关键代码：

- 获取参数名称
- 获取参数信息，如类型之类的
- 进行具体参数值
- 进行参数转换

其中参数转换还比较麻烦，粗略看了下细节还是比较多，本地先跳过，后面再过来看

我们继续跟踪获取具体参数值的相关函数： resolveName(resolvedName.toString(), nestedParameter, webRequest);

```java
	# RequestParamMethodArgumentResolver.java
	@Override
	@Nullable
	protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
		HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);

		if (servletRequest != null) {
			Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
			if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
				return mpArg;
			}
		}

		Object arg = null;
		// 看着像是获取文件？
		MultipartRequest multipartRequest = request.getNativeRequest(MultipartRequest.class);
		if (multipartRequest != null) {
			List<MultipartFile> files = multipartRequest.getFiles(name);
			if (!files.isEmpty()) {
				arg = (files.size() == 1 ? files.get(0) : files);
			}
		}
		if (arg == null) {
			// 获取参数名中请求中对应的值
			// http://localhost:8080?id=1&id=2 就得到了 [1, 2]
			String[] paramValues = request.getParameterValues(name);
			if (paramValues != null) {
				arg = (paramValues.length == 1 ? paramValues[0] : paramValues);
			}
		}
		return arg;
	}
```

在上面的代码中，我们可以看到参数值的获取是先去 multipartRequest 中获取，如果为空再到函数： getParameterValues 中获取

看得好像是文件优先？是不是同名的话，就优先获取文件了？这个大家也可以试下

继续跟踪函数： getParameterValues

```java
    # Parameters.java
    public String[] getParameterValues(String name) {
        handleQueryParameters();
        // no "facade"
        ArrayList<String> values = paramHashValues.get(name);
        if (values == null) {
            return null;
        }
        return values.toArray(new String[0]);
    }
```

跟踪下去就是Tomcat的函数处理了，感觉很有类似自己写Tcp服务时自定义解析协议的感觉，这部分细节就不细看了

到这里就得到最终的结果： [1, 2]

经过： arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter); 得到了最后的值： 1

## 总结
本篇文章中，我们大致探索了请求参数的获取源码，大致有下面的步骤：

- 获取所需的参数列表，其中有一些参数的信息，比如类型，进行遍历获取其值
- findProvidedArgument 匹配获取，这部分功能未探索
- this.resolvers.resolveArgument :参数解析器获取
  - 遍历所有的解析器
  - 得到对应的解析器
  - 得到对应的值
  - 值的转换

期间，我们实验两种请求：

```sh
GET http://localhost:8080/?id=1&id=2

POST http://localhost:8080/test2
Content-Type: application/json

{"name": "name", "password": "password"}
```

两个请求的参数解析器是不一样的，但也是我们日常开发中常用的，比如第二个，但本篇文章中并没有具体解析，大家感兴趣可以自行根据本篇的思路进行解析

## 参考链接
- [Spring 源码解析系列](https://juejin.cn/post/6983145193117581343)