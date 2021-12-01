

# Spring

***

- 
- 0 ProxyingHandlerMethodArgumentResolver
- 1 RequestParamMethodArgumentResolver : [springMVC源码分析--RequestParamMethodArgumentResolver参数解析器（三）](https://blog.csdn.net/qq924862077/article/details/54292082)
- 2 RequestParamMapMethodArgumentResolver : [springmvc的RequestParamMapMethodArgumentResolver分析](http://blog.sina.com.cn/s/blog_e660c25b01030ib8.html)
- 3 PathVariableMethodArgumentResolver : [Spring 注解面面通 之 @PathVariable参数绑定源码解析](https://blog.csdn.net/securitit/article/details/110207357)
- 4 PathVariableMapMethodArgumentResolver  : [Spring 注解面面通 之 @PathVariable参数绑定源码解析](https://blog.csdn.net/securitit/article/details/110207357)
- 5 MatrixVariableMethodArgumentResolver
- 6 MatrixVariableMapMethodArgumentResolver
- 7 ServletModelAttributeMethodProcessor
- 8 RequestResponseBodyMethodProcessor
- 9 RequestPartMethodArgumentResolver
- 10 RequestHeaderMethodArgumentResolver
- 11 RequestHeaderMapMethodArgumentResolver
- 12 ServletCookieValueMethodArgumentResolver
- 13 ExpressionValueMethodArgumentResolver
- 14 SessionAttributeMethodArgumentResolver
- 15 RequestAttributeMethodArgumentResolver
- 16 ServletRequestMethodArgumentResolver
- 17 ServletResponseMethodArgumentResolver
- 18 HttpEntityMethodProcessor
- 19 RedirectAttributesMethodArgumentResolver
- 20 ModelMethodProcessor
- 21 MapMethodProcessor
- 22 ErrorsMethodArgumentResolver
- 23 SessionStatusMethodArgumentResolver
- 24 UriComponentsBuilderMethodArgumentResolver
- 25 AuthenticationPrincipalArgumentResolver
- 26 AuthenticationPrincipalArgumentResolver
- 27 CurrentSecurityContextArgumentResolver
- 28 CsrfTokenArgumentResolver
- 29 SortHandlerMethodArgumentResolver
- 30 PageableHandlerMethodArgumentResolver
- 31 ProxyingHandlerMethodArgumentResolver
- 32 RequestParamMethodArgumentResolver
- 33 ServletModelAttributeMethodProcessor



### 基于注解的参数解析

基于注解的参数解析 <-- 解析的数据来源主要是 HttpServletRequest | ModelAndViewContainer

Annotation-based argument resolution



- RequestParamMethodArgumentResolver
  - 解析被注解 @RequestParam, @RequestPart 修饰的参数, 数据的获取通过 HttpServletRequest.getParameterValues
- RequestParamMapMethodArgumentResolver
  - 解析被注解 @RequestParam 修饰, 且类型是 Map 的参数, 数据的获取通过 HttpServletRequest.getParameterMap
- PathVariableMethodArgumentResolver
  - 解析被注解 @PathVariable 修饰, 数据的获取通过 uriTemplateVars, 而 uriTemplateVars 却是通过 RequestMappingInfoHandlerMapping.handleMatch 生成, 其实就是 uri 中映射出的 key <-> value
- PathVariableMapMethodArgumentResolver
  - 解析被注解 @PathVariable 修饰 且数据类型是 Map, 数据的获取通过 uriTemplateVars, 而 uriTemplateVars 却是通过 RequestMappingInfoHandlerMapping.handleMatch 生成, 其实就是 uri 中映射出的 key <-> value
- MatrixVariableMethodArgumentResolver
  - 解析被注解 @MatrixVariable 修饰, 数据的获取通过 URI提取了;后存储的 uri template 变量值
- MatrixVariableMapMethodArgumentResolver
  - 解析被注解 @MatrixVariable 修饰 且数据类型是 Map, 数据的获取通过 URI提取了;后存储的 uri template 变量值
- ServletModelAttributeMethodProcessor
  - 解析被注解 @ModelAttribute 修饰, 且类型是 Map 的参数, 数据的获取通过 ModelAndViewContainer 获取, 通过 DataBinder 进行绑定
- RequestResponseBodyMethodProcessor
  - 解析被注解 @RequestBody 修饰的参数, 以及被@ResponseBody修饰的返回值, 数据的获取通过 HttpServletRequest 获取, 根据 MediaType通过HttpMessageConverter转换成对应的格式, 在处理返回值时 也是通过 MediaType 选择合适HttpMessageConverter, 进行转换格式, 并输出
- RequestPartMethodArgumentResolver
  - 解析被注解 @RequestPart 修饰, 数据的获取通过 HttpServletRequest.getParts()
- RequestHeaderMethodArgumentResolver
  - 解析被注解 @RequestHeader 修饰, 数据的获取通过 HttpServletRequest.getHeaderValues()
- RequestHeaderMapMethodArgumentResolver
  - 解析被注解 @RequestHeader 修饰且参数类型是 Map, 数据的获取通过 HttpServletRequest.getHeaderValues()
- ServletCookieValueMethodArgumentResolver
  - 解析被注解 @CookieValue 修饰, 数据的获取通过 HttpServletRequest.getCookies()
- ExpressionValueMethodArgumentResolver
  - 解析被注解 @Value 修饰, 数据在这里没有解析
- SessionAttributeMethodArgumentResolver
  - 解析被注解 @SessionAttribute 修饰, 数据的获取通过 HttpServletRequest.getAttribute(name, RequestAttributes.SCOPE_SESSION)
- RequestAttributeMethodArgumentResolver
  - 解析被注解 @RequestAttribute 修饰, 数据的获取通过 HttpServletRequest.getAttribute(name, RequestAttributes.SCOPE_REQUEST)



### 基于类型的参数解析（Type-based argument resolution）

- ServletRequestMethodArgumentResolver
  - 解析固定类型参数(比如: ServletRequest, HttpSession, InputStream 等), 参数的数据获取还是通过 HttpServletRequest
- ServletResponseMethodArgumentResolver
  - 解析固定类型参数(比如: ServletResponse, OutputStream等), 参数的数据获取还是通过 HttpServletResponse
- HttpEntityMethodProcessor
  - 解析固定类型参数(比如: HttpEntity, RequestEntity 等), 参数的数据获取还是通过 HttpServletRequest
- RedirectAttributesMethodArgumentResolver
  - 解析固定类型参数(比如: RedirectAttributes), 参数的数据获取还是通过 HttpServletResponse
- ModelMethodProcessor
  - 解析固定类型参数(比如: Model等), 参数的数据获取通过 ModelAndViewContainer
- ErrorsMethodArgumentResolver
  - 解析固定类型参数(比如: Errors), 参数的数据获取通过 ModelAndViewContainer
- SessionStatusMethodArgumentResolver
  - 解析固定类型参数(比如: SessionStatus), 参数的数据获取通过 ModelAndViewContainer
- UriComponentsBuilderMethodArgumentResolver
  - 解析固定类型参数(比如: UriComponentsBuilder), 参数的数据获取通过 HttpServletRequest



### 自定义参数解析器（Custom arguments）

```javascript
// 3.自定义参数解析器
// Custom arguments
if (getCustomArgumentResolvers() != null) {
    resolvers.addAll(getCustomArgumentResolvers());
}
```



### 特殊的两个解析器

```java
// Catch-all
//这两个解析器可以解析所有类型的参数
resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
resolvers.add(new ServletModelAttributeMethodProcessor(true));
```



## 参考链接

- [Spring参数的自解析--还在自己转换？你out了!](https://cloud.tencent.com/developer/article/1490089)