# Spring源码解析系列：Spring Web 请求初探

---
## 简介
这几年工作中大部分时间都与SpringWeb打交道，前几年源码阅读能力和方法论不行，在疫情期间学习了一波，有了源码阅读与分析能力，目前也有了一些闲暇时间，以目前自己的水平去写写Spring相关的源码解析

本篇文章就先简单热个身，看看一个请求是如何处理到达我们平时写的Controllers里面的

## 准备工作
准备工作较为简单，我们使用 [spring initializr](https://start.spring.io/),搭建一个初始工程，写一个简单的HelloWorld

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloWorld {

    @GetMapping("/")
    public String helloWorld() {
        return "Hello world";
    }
}
```

推荐IDEA上一个比较好用的插件：RestfulTool和RestfulToolKit

启动项目后，会读取所有的请求列表，简单快捷的发起一个请求：

![IDEA推荐SpringWeb插件.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a0797804ce8847d598ed710345aa3c70~tplv-k3u1fbpfcp-watermark.image)

这样，本章的初步环境就搭建好了，下面开始源码分析：

## 源码解析
我们直接在新增的HelloWorld类的：

```java
return "Hello world"; 
```

这行打断点，直接发起请求，可以看到下面的调用栈：

![调试调用栈.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ee8b661927549cc97369f4eea940f66~tplv-k3u1fbpfcp-watermark.image)

我们从下网上，大体的点击进去，看看大意

带着问题去看：

- 从那接收的请求：如何写过相关网络编程的，那就会想到起码有一个Netty之类的服务，监听在指定端口，接收请求
- 接收到的请求如何找到我们写的页面代码（HelloWorld）：比如如何将请求路径和处理函数进行对应

一个简单的本地探索的请求如下：

![Springweb初探.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4687e652cc7f41b5bd6a6be4dfe43511~tplv-k3u1fbpfcp-watermark.image)

### 找到大致的监听入口
我们从下往上看是，发现有一个类：NioEndpoint.java,虽然看不太懂更多的细节，但知道有这么一个东西即可

### 找到重要的Request和Response处理函数
我们继续来到一个重要的类：Http11Processor.java，其中有添加Filter相关的代码，还是明显的Request和Response的处理

这里我们留下两个疑问，目前我们先把请求路径梳理下，这些细节日后我们再回过头来啃它

- 这个类中设置的Filter如何使用
- Request和Response是如何转化得到的

```java
public class Http11Processor extends AbstractProcessor {
    @Override
    public SocketState service(SocketWrapperBase<?> socketWrapper)
        throws IOException {
        RequestInfo rp = request.getRequestProcessor();
        rp.setStage(org.apache.coyote.Constants.STAGE_PARSE);

        // Setting up the I/O
        setSocketWrapper(socketWrapper);

        // Flags
        keepAlive = true;
        openSocket = false;
        readComplete = true;
        boolean keptAlive = false;
        SendfileState sendfileState = SendfileState.DONE;

        while (!getErrorState().isError() && keepAlive && !isAsync() && upgradeToken == null &&
                sendfileState == SendfileState.DONE && !protocol.isPaused()) {

            // Parsing the request header
            try {
		......
            }

            // Has an upgrade been requested?
            if (isConnectionToken(request.getMimeHeaders(), "upgrade")) {
                ......
            }
	    ......

            // Process the request in the adapter
            if (getErrorState().isIoAllowed()) {
                try {
                    rp.setStage(org.apache.coyote.Constants.STAGE_SERVICE);
                    getAdapter().service(request, response);
                    // Handle when the response was committed before a serious
                    // error occurred.  Throwing a ServletException should both
                    // set the status to 500 and set the errorException.
                    // If we fail here, then the response is likely already
                    // committed, so we can't try and set headers.
                    if(keepAlive && !getErrorState().isError() && !isAsync() &&
                            statusDropsConnection(response.getStatus())) {
                        setErrorState(ErrorState.CLOSE_CLEAN, null);
                    }
                } catch (InterruptedIOException e) {
                    setErrorState(ErrorState.CLOSE_CONNECTION_NOW, e);
                } catch (HeadersTooLargeException e) {
                    log.error(sm.getString("http11processor.request.process"), e);
                    // The response should not have been committed but check it
                    // anyway to be safe
                    if (response.isCommitted()) {
                        setErrorState(ErrorState.CLOSE_NOW, e);
                    } else {
                        response.reset();
                        response.setStatus(500);
                        setErrorState(ErrorState.CLOSE_CLEAN, e);
                        response.setHeader("Connection", "close"); // TODO: Remove
                    }
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    log.error(sm.getString("http11processor.request.process"), t);
                    // 500 - Internal Server Error
                    response.setStatus(500);
                    setErrorState(ErrorState.CLOSE_CLEAN, t);
                    getAdapter().log(request, response, 0);
                }
            }
        }
	......
    }
}
```

### 经过了一系列的Valva类处理
继续往上查看，我们看到了很多的Value相关的处理，类型一种责任链的处理方式，但目前还不知道其具体作用，但没关系，我们继续往下看

### 经过了一系列的Filter处理
经过上面那些Valva类的处理后，来到了我们熟悉的Filter处理，具体细节我们先不深究，继续往下，留下疑问待日后处理

- Filter如何初始化的
- 如何针对指定的请求路径进行处理
- 自定义添加的Filter会插入其中吗？如何插入？

### 请求路径到具体函数处理
继续走，我们来到属性的类：DispatcherServlet.java，平时看文章之类的，这个类的出现率还是挺高的

通过断点调试，我们发现在这里直接得到了我们请求路径对应的请求方法，如下代码中标注的：

```java
public class DispatcherServlet {
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	boolean multipartRequestParsed = false;

	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

	try {
		ModelAndView mv = null;
		Exception dispatchException = null;

		try {
			processedRequest = checkMultipart(request);
			multipartRequestParsed = (processedRequest != request);

			// Determine handler for the current request.
			// 得到请求的处理方法
			mappedHandler = getHandler(processedRequest);
			if (mappedHandler == null) {
				noHandlerFound(processedRequest, response);
				return;
			}

			if (!mappedHandler.applyPreHandle(processedRequest, response)) {
				return;
			}

			// Actually invoke the handler.
			// 这行通过断点查看：mappedHandler就是我们请求的处理方法
			mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

			if (asyncManager.isConcurrentHandlingStarted()) {
				return;
			}

			applyDefaultViewName(processedRequest, mv);
			mappedHandler.applyPostHandle(processedRequest, response, mv);
		}
		catch (Exception ex) {
			........
		}
	}
}

```

我们继续往下，来到了一个类：InvocableHandlerMethod.java，其中的：method(getBean(), args)，相比很熟悉了，经典的反射调用

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

在这里我们也留下疑问：

- 如何得到请求路径对应的处理方法？
- 如何解析获取处理方法的参数的？

## 总结
经过上面的简单分析，我们大致得到了一个请求的处理路径：

- NioEndpoint.java : 服务监听
- Http11Processor.java : 请求与响应处理入口
- 一系列的Valva类处理
- 一系列的Filter处理
- DispatcherServlet.java : 请求路径到具体处理方法
- InvocableHandlerMethod.java : 反射调用处理

当前我们拿到一个比较大的地图，但很多地方还被迷雾笼罩，接下来我们慢慢去除迷雾，探索其奥秘