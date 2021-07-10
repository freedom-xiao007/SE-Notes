# Spring源码解析 -- SpringWeb请求映射Map初始化
***
## 简介
在上篇文章中，大致解析了Spring如何将请求路径与处理方法进行映射，但映射相关的初始化对于我们来说还是一团迷雾

本篇文章就来探索下，请求路径和处理方法的映射，是如何进行初始化的

## 概览
基于上篇文章：[Spring 源码解析 -- SpringWeb请求映射解析](https://juejin.cn/post/6980874051669458952)

本篇文章本来想早点写完，但一直卡着，没有找到想要的切入点，还好在周四左右定位到了相关的关键代码，初步探索到相关初始化的代码过程

接下来就展示这段时间的定位和解析过程，下面是自己这段时间探索历程：

- 想定位 handlerMappings 的初始化，但没有定位请求URL和处理方法相关初始化的东西
- 回过来头来看 handlerMappings ，看其有哪些东西,发现这个Map中并没有自定义的HelloWorld
- 意识到关键的 RequestMappingHandlerMapping，跟踪发送只有在这个类型才成功匹配
- 回顾上篇的请求映射解析，在 RequestMappingHandlerMapping 细致初始化相关的代码
- 成功找到相关的路径和处理方法初始化的关键代码

接下来详细看下：

## 源码解析
### 初步探索初始化：误入歧途
在类： DispatcherServlet.java 中，我们定位到 mappedHandler 获取的关键代码

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

它就是遍历了： handlerMappings ,于是去跟踪了 handlerMappings 初始化过程

结果失败而归，没有找到自己想要的东西，并没有发现自定义类： HelloWorld 的相关东西

这块有很多的代码，里面也有很多的东西，但在这里就不展示出来了，感兴趣的老哥可自行探索

### 回顾请求映射查找匹配：幡然醒悟
探索 handlerMappings 无果，于是回到上面那段遍历处理

经过调试 handlerMappings 基本是固定的，包含下面的类：

- this.handlerMappings
    - RequestMappingHandlerMapping
    - BeanNameUrlHandlerMapping
    - RouterFunctionMapping
    - SimpleUrlHandlerMapping
    - WelcomePageHandlerMapping

而匹配的成功的是： RequestMappingHandlerMapping ，其返回了我们想要的处理方法 HelloWorld

调试中很疑惑为啥初期要初始化这几个了，并且再套了一层请求匹配，目前掌握的知识还不足于破解，只能后面再探索了

于是开始梳理 RequestMappingHandlerMapping 的请求匹配，在下面的一段关键代码匹配成功了：

```java
    # AbstractHandlerMethodMapping.java
    @Nullable
	protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
		List<Match> matches = new ArrayList<>();
        // 在这里拿到了匹配结果
		List<T> directPathMatches = this.mappingRegistry.getMappingsByDirectPath(lookupPath);
		if (directPathMatches != null) {
			addMatchingMappings(directPathMatches, matches, request);
		}
		if (matches.isEmpty()) {
			addMatchingMappings(this.mappingRegistry.getRegistrations().keySet(), matches, request);
		}
		.......
	}
```

在上面的代码中就匹配成功了，其中匹配的方法很简单粗暴：

```java
    # AbstractHandlerMethodMapping.java -- MappingRegistry
    @Nullable
	public List<T> getMappingsByDirectPath(String urlPath) {
		return this.pathLookup.get(urlPath);
	}
```

于是关键点到了 this.mappingRegistry 的初始化，找到初始化的代码，打上断点

期间以为是在类：AbstractHandlerMethodMapping 中进行的初始的，在下面的函数打上了断点：

```java
    # AbstractHandlerMethodMapping.java
    public void setPatternParser(PathPatternParser patternParser) {
		Assert.state(this.mappingRegistry.getRegistrations().isEmpty(),
				"PathPatternParser must be set before the initialization of " +
						"request mappings through InitializingBean#afterPropertiesSet.");
		super.setPatternParser(patternParser);
	}

    public void registerMapping(T mapping, Object handler, Method method) {
		if (logger.isTraceEnabled()) {
			logger.trace("Register \"" + mapping + "\" to " + method.toGenericString());
		}
		this.mappingRegistry.register(mapping, handler, method);
	}
```

但一直进不去，于是直接在其定义的内部类中： MappingRegistry 中进行寻找，并成功定位到想要的关键代码

### 请求映射关键代码定位：柳暗花明
阅读类： MappingRegistry 的相关代码，发现下面的方法和可以，我们直接打上断点，重启程序：

发现了前面的: this.pathLookup 的相关添加操作

```java
    public void register(T mapping, Object handler, Method method) {
		this.readWriteLock.writeLock().lock();
		try {
			HandlerMethod handlerMethod = createHandlerMethod(handler, method);
			validateMethodMapping(handlerMethod, mapping);

			Set<String> directPaths = AbstractHandlerMethodMapping.this.getDirectPaths(mapping);
			for (String path : directPaths) {
                // 这段代码是关键
				this.pathLookup.add(path, mapping);
			}

			String name = null;
			if (getNamingStrategy() != null) {
				name = getNamingStrategy().getName(handlerMethod, mapping);
				addMappingName(name, handlerMethod);
			}

			CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
			if (corsConfig != null) {
				corsConfig.validateAllowCredentials();
				this.corsLookup.put(handlerMethod, corsConfig);
			}

			this.registry.put(mapping,
				new MappingRegistration<>(mapping, handlerMethod, directPaths, name, corsConfig != null));
			}
		finally {
			this.readWriteLock.writeLock().unlock();
		}
    }
```

应用重启后，果然顺利来到我们打上的断点处，通过分析调用栈，我们确实找到了请求映射的关键代码

我们将调用栈从下网上分析查看：

#### 应用启动相关
开始就是熟悉Spring启动相关，这些代码相信大家尝试阅读源码的时候读过很多遍了

跟踪发现在： DefaultListableBeanFactory.class 的 preInstantiateSingletons 方法中个，一大段嵌套循环，心想这段代码目前能优化吗？

```java
    public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
        return (new SpringApplication(primarySources)).run(args);
    }

    public static void main(String[] args) throws Exception {
        run(new Class[0], args);
    }

    public ConfigurableApplicationContext run(String... args) {
        ......
        try {
            ......
            this.prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
            // 从下面这个进入
            this.refreshContext(context);
            this.afterRefresh(context, applicationArguments);
            ......
        } catch (Throwable var10) {
            ......
            this.handleRunFailure(context, var10, listeners);
            throw new IllegalStateException(var10);
        }
        ......
    }

    public void refresh() throws BeansException, IllegalStateException {
        synchronized(this.startupShutdownMonitor) {
            .......
            try {
                this.postProcessBeanFactory(beanFactory);
                StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
                this.invokeBeanFactoryPostProcessors(beanFactory);
                this.registerBeanPostProcessors(beanFactory);
                beanPostProcess.end();
                this.initMessageSource();
                this.initApplicationEventMulticaster();
                this.onRefresh();
                this.registerListeners();
                // 从这里进入
                this.finishBeanFactoryInitialization(beanFactory);
                this.finishRefresh();
            } catch (BeansException var10) {
            } finally {
                ......
            }
            ......
        }
    }
```

#### RequestMappingHandlerMapping 相关初始化
继续跟踪下面的，看到了属性的CreateBean和afterPropertiesSet

```java
    # AbstractAutowireCapableBeanFactory.class
    protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
        ......
        try {
            // 这里初始化了 RequestMappingHandlerMapping
            beanInstance = this.doCreateBean(beanName, mbdToUse, args);
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Finished creating instance of bean '" + beanName + "'");
            }

            return beanInstance;
        } catch (ImplicitlyAppearedSingletonException | BeanCreationException var7) {
            throw var7;
        } catch (Throwable var8) {
            throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", var8);
        }
    }

    # AbstractAutowireCapableBeanFactory.class
    protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd) throws Throwable {
        boolean isInitializingBean = bean instanceof InitializingBean;
        if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
            }

            if (System.getSecurityManager() != null) {
                try {
                    AccessController.doPrivileged(() -> {
                        ((InitializingBean)bean).afterPropertiesSet();
                        return null;
                    }, this.getAccessControlContext());
                } catch (PrivilegedActionException var6) {
                    throw var6.getException();
                }
            } else {
                // 这里进入请求映射的相关操作
                ((InitializingBean)bean).afterPropertiesSet();
            }
        }
        ......
    }
```

#### 请求映射初始化
继续跟踪下去，看看了循环遍历Controllers相关的代码（还有很多细节没搞清，后面再继续了，先梳理主线）

```java
    # AbstractHandlerMethodMapping.java
    @Override
	public void afterPropertiesSet() {
        // 初始化请求映射
		initHandlerMethods();
	}

    protected void initHandlerMethods() {
        // 遍历所有的自定义的Controllers，后面自己又定义了一个Controllers
		for (String beanName : getCandidateBeanNames()) {
			if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
                // 在这里看到了我们定义的HelloWorld
				processCandidateBean(beanName);
			}
		}
		handlerMethodsInitialized(getHandlerMethods());
	}

    protected String[] getCandidateBeanNames() {
		return (this.detectHandlerMethodsInAncestorContexts ?
				BeanFactoryUtils.beanNamesForTypeIncludingAncestors(obtainApplicationContext(), Object.class) :
				obtainApplicationContext().getBeanNamesForType(Object.class));
	}
```

继续跟踪下去，看到了下面的获取类中具体请求路径相关的代码，并且到了具体的初始化请求映射的代码

```java
    # AbstractHandlerMethodMapping.java
    protected void processCandidateBean(String beanName) {
		Class<?> beanType = null;
		try {
			beanType = obtainApplicationContext().getType(beanName);
		}
		catch (Throwable ex) {
			// An unresolvable bean type, probably from a lazy bean - let's ignore it.
			if (logger.isTraceEnabled()) {
				logger.trace("Could not resolve type for bean '" + beanName + "'", ex);
			}
		}
		if (beanType != null && isHandler(beanType)) {
            // 得到Controller Bean后的入口
			detectHandlerMethods(beanName);
		}
	}

    protected void detectHandlerMethods(Object handler) {
		Class<?> handlerType = (handler instanceof String ?
				obtainApplicationContext().getType((String) handler) : handler.getClass());

		if (handlerType != null) {
            // 处理得到所有的Controllers method
			Class<?> userType = ClassUtils.getUserClass(handlerType);
			Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
					(MethodIntrospector.MetadataLookup<T>) method -> {
						try {
							return getMappingForMethod(method, userType);
						}
						catch (Throwable ex) {
							throw new IllegalStateException("Invalid mapping on handler class [" +
									userType.getName() + "]: " + method, ex);
						}
					});
			if (logger.isTraceEnabled()) {
				logger.trace(formatMappings(userType, methods));
			}
			else if (mappingsLogger.isDebugEnabled()) {
				mappingsLogger.debug(formatMappings(userType, methods));
			}
			methods.forEach((method, mapping) -> {
				Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
                // 注册
				registerHandlerMethod(handler, invocableMethod, mapping);
			});
		}
	}

    public void register(T mapping, Object handler, Method method) {
		this.readWriteLock.writeLock().lock();
		try {
    		HandlerMethod handlerMethod = createHandlerMethod(handler, method);
			validateMethodMapping(handlerMethod, mapping);

			Set<String> directPaths = AbstractHandlerMethodMapping.this.getDirectPaths(mapping);
			for (String path : directPaths) {
                // 映射添加
				this.pathLookup.add(path, mapping);
			}
		}
		finally {
			this.readWriteLock.writeLock().unlock();
		}
	}
```

## 总结
经过一段时间的探索的整理，我们终于得到了大致的请求路径映射初始化的代码

- 1.应用启动时，初始化：RequestMappingHandlerMapping
- 2.RequestMappingHandlerMapping 中请求路径初始化

经过调试，我们还发现虽然 RequestMappingHandlerMapping 是一开始就初始化了，但加载到 handlerMappings 是第一次请求的时候才加载进去的

本篇虽然得到了大致的请求路径初始化的代码，但其中有很多细节是值得探索的，比如Bean中Method的处理

之前自己写过一些DI和Web相关的Demo，停在了Servlet，卡在了请求映射初始化和匹配，这个给了我一些思路，后面详细看看这块代码，完善下之前的Demo

## 参考链接
- [Spring 源码解析系列](https://juejin.cn/post/6983145193117581343)