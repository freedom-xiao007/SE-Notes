# Soul网关源码解析（八）路由匹配初探
***
## 简介
&ensp;&ensp;&ensp;&ensp;&ensp;今日看看路由的匹配相关代码，查看HTTP的DividePlugin匹配

## 示例运行
&ensp;&ensp;&ensp;&ensp;&ensp;使用HTTP的示例，运行Soul-Admin，Soul-Bootstrap，Soul-Example-HTTP

&ensp;&ensp;&ensp;&ensp;&ensp;记得启动数据库

```shell script
docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:latest
```

&ensp;&ensp;&ensp;&ensp;&ensp;其他的就不再赘述了，有问题可以参照前面的文章，看看有没有啥借鉴的

## 源码Debug
&ensp;&ensp;&ensp;&ensp;&ensp;在番外篇：[Soul网关源码阅读番外篇（一） HTTP参数请求错误](https://juejin.cn/post/6918947689564471309)，我们知道了GlobalPlugin的重要性，其会将请求对应的真实是后台服务器路径写入Exchange中，我们先来摸一摸其具体细节：

&ensp;&ensp;&ensp;&ensp;&ensp;首先打上在类的execute中打上断点，访问：http://127.0.0.1:9195/http/order/findById?id=1111

&ensp;&ensp;&ensp;&ensp;&ensp;进入断点后，继续跟入

```java
public class GlobalPlugin implements SoulPlugin {
    ......
    
    @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        final ServerHttpRequest request = exchange.getRequest();
        final HttpHeaders headers = request.getHeaders();
        final String upgrade = headers.getFirst("Upgrade");
        SoulContext soulContext;
        if (StringUtils.isBlank(upgrade) || !"websocket".equals(upgrade)) {
            // 进入build函数，进行操作
            soulContext = builder.build(exchange);
        } else {
            final MultiValueMap<String, String> queryParams = request.getQueryParams();
            soulContext = transformMap(queryParams);
        }
        exchange.getAttributes().put(Constants.CONTEXT, soulContext);
        return chain.execute(exchange);
    }
    ......
}
```

&ensp;&ensp;&ensp;&ensp;&ensp;跟进build里面去，里面首先获取了路径，进行请求类型判断，没有元数据则走到了默认的HTTP

```java
public class DefaultSoulContextBuilder implements SoulContextBuilder {
    
    @Override
    public SoulContext build(final ServerWebExchange exchange) {
        final ServerHttpRequest request = exchange.getRequest();
        // path = /http/order/findById
        String path = request.getURI().getPath();
        MetaData metaData = MetaDataCache.getInstance().obtain(path);
        if (Objects.nonNull(metaData) && metaData.getEnabled()) {
            exchange.getAttributes().put(Constants.META_DATA, metaData);
        }
        // 进入 transform 函数
        return transform(request, metaData);
    }
    
    private SoulContext transform(final ServerHttpRequest request, final MetaData metaData) {
        final String appKey = request.getHeaders().getFirst(Constants.APP_KEY);
        final String sign = request.getHeaders().getFirst(Constants.SIGN);
        final String timestamp = request.getHeaders().getFirst(Constants.TIMESTAMP);
        SoulContext soulContext = new SoulContext();
        // path = /http/order/findById
        String path = request.getURI().getPath();
        soulContext.setPath(path);
        if (Objects.nonNull(metaData) && metaData.getEnabled()) {
            if (RpcTypeEnum.SPRING_CLOUD.getName().equals(metaData.getRpcType())) {
                setSoulContextByHttp(soulContext, path);
                soulContext.setRpcType(metaData.getRpcType());
            } else if (RpcTypeEnum.DUBBO.getName().equals(metaData.getRpcType())) {
                setSoulContextByDubbo(soulContext, metaData);
            } else if (RpcTypeEnum.SOFA.getName().equals(metaData.getRpcType())) {
                setSoulContextBySofa(soulContext, metaData);
            } else if (RpcTypeEnum.TARS.getName().equals(metaData.getRpcType())) {
                setSoulContextByTars(soulContext, metaData);
            } else {
                setSoulContextByHttp(soulContext, path);
                soulContext.setRpcType(RpcTypeEnum.HTTP.getName());
            }
        } else {
            // 来打这，进行HTTP设置
            setSoulContextByHttp(soulContext, path);
            soulContext.setRpcType(RpcTypeEnum.HTTP.getName());
        }
        soulContext.setAppKey(appKey);
        soulContext.setSign(sign);
        soulContext.setTimestamp(timestamp);
        soulContext.setStartDateTime(LocalDateTime.now());
        Optional.ofNullable(request.getMethod()).ifPresent(httpMethod -> soulContext.setHttpMethod(httpMethod.name()));
        return soulContext;
    }
    
    private void setSoulContextByHttp(final SoulContext soulContext, final String path) {
        String contextPath = "/";
        // 是一个列表，值是：http, order, findById
        String[] splitList = StringUtils.split(path, "/");
        if (splitList.length != 0) {
            // 这个应该是前缀的意思，并且只取第一个，值是：/http
            contextPath = contextPath.concat(splitList[0]);
        }
        // 取后面的字符串，得到：/order/findById
        String realUrl = path.substring(contextPath.length());
        soulContext.setContextPath(contextPath);
        soulContext.setModule(contextPath);
        soulContext.setMethod(realUrl);
        soulContext.setRealUrl(realUrl);
    }
}
```

&ensp;&ensp;&ensp;&ensp;&ensp;在最后一个函数中，我们看到了具体设置realURL的代码，其大致思路，如上面代码总描述的一样

&ensp;&ensp;&ensp;&ensp;&ensp;这里就有个小疑问，前缀也就是只能是 /xxx 之类的，如果是 /xxx/xxx 那请求后面是否会出问题

&ensp;&ensp;&ensp;&ensp;&ensp;我们做了一个小实验，设置一个选择器为条件为：/more/prefix,一个规则为：/more/prefix/baidu，都是相等条件

&ensp;&ensp;&ensp;&ensp;&ensp;下面Debug来看下GlobalPlugin的解析结果，如下，明显不是我们想要的，所有这里初步猜测不能选择器不能使用两级前缀，不然可能会出问题

```
contextPath = /more
realURL = /prefix/baidu
```

&ensp;&ensp;&ensp;&ensp;&ensp;下面我继续看下，DividePlugin的匹配详情，首先打入断点在 AbstractSoulPlugin,执行匹配逻辑

```java
public abstract class AbstractSoulPlugin implements SoulPlugin {
    ......
    @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        String pluginName = named();
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
        if (pluginData != null && pluginData.getEnabled()) {
            final Collection<SelectorData> selectors = BaseDataCache.getInstance().obtainSelectorData(pluginName);
            if (CollectionUtils.isEmpty(selectors)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            // use /http/order/findById
            // 这里首先进行选择器的匹配，我们看下选择器如果的匹配细节
            final SelectorData selectorData = matchSelector(exchange, selectors);
            if (Objects.isNull(selectorData)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            selectorLog(selectorData, pluginName);
            final List<RuleData> rules = BaseDataCache.getInstance().obtainRuleData(selectorData.getId());
            if (CollectionUtils.isEmpty(rules)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            RuleData rule;
            if (selectorData.getType() == SelectorTypeEnum.FULL_FLOW.getCode()) {
                //get last
                rule = rules.get(rules.size() - 1);
            } else {
                rule = matchRule(exchange, rules);
            }
            if (Objects.isNull(rule)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            ruleLog(rule, pluginName);
            return doExecute(exchange, chain, selectorData, rule);
        }
        return chain.execute(exchange);
    }

    private SelectorData matchSelector(final ServerWebExchange exchange, final Collection<SelectorData> selectors) {
        // 循环每个选择器，看是否能匹配得上，findFirst的意思是否多个匹配上就要第一个？
        return selectors.stream()
                .filter(selector -> selector.getEnabled() && filterSelector(selector, exchange))
                .findFirst().orElse(null);
    }

    private Boolean filterSelector(final SelectorData selector, final ServerWebExchange exchange) {
        if (selector.getType() == SelectorTypeEnum.CUSTOM_FLOW.getCode()) {
            if (CollectionUtils.isEmpty(selector.getConditionList())) {
                return false;
            }
            // 使用匹配策略工具进行匹配，我们进行跟下去
            return MatchStrategyUtils.match(selector.getMatchMode(), selector.getConditionList(), exchange);
        }
        return true;
    }

    private RuleData matchRule(final ServerWebExchange exchange, final Collection<RuleData> rules) {
        return rules.stream()
                .filter(rule -> filterRule(rule, exchange))
                .findFirst().orElse(null);
    }

    private Boolean filterRule(final RuleData ruleData, final ServerWebExchange exchange) {
        return ruleData.getEnabled() && MatchStrategyUtils.match(ruleData.getMatchMode(), ruleData.getConditionDataList(), exchange);
    }
    ......
}
```

&ensp;&ensp;&ensp;&ensp;&ensp;继续跟到匹配策略工具的类中，它有and和or的匹配策略，判断策略，构造相关策略类后进行调用

```java
public class MatchStrategyUtils {

    public static boolean match(final Integer strategy, final List<ConditionData> conditionDataList, final ServerWebExchange exchange) {
        // and 策略，构造and策略类，进行匹配；继续跟进match
        String matchMode = MatchModeEnum.getMatchModeByCode(strategy);
        MatchStrategy matchStrategy = ExtensionLoader.getExtensionLoader(MatchStrategy.class).getJoin(matchMode);
        return matchStrategy.match(conditionDataList, exchange);
    }
}
```

&ensp;&ensp;&ensp;&ensp;&ensp;进行跟到judge函数中

```java
public class AndMatchStrategy extends AbstractMatchStrategy implements MatchStrategy {

    @Override
    public Boolean match(final List<ConditionData> conditionDataList, final ServerWebExchange exchange) {
        return conditionDataList
                .stream()
                .allMatch(condition -> OperatorJudgeFactory.judge(condition, buildRealData(condition, exchange)));
    }
}
```

&ensp;&ensp;&ensp;&ensp;&ensp;再根据judge，有点复杂感觉......

```java
public class OperatorJudgeFactory {

    public static Boolean judge(final ConditionData conditionData, final String realData) {
        if (Objects.isNull(conditionData) || StringUtils.isBlank(realData)) {
            return false;
        }
        return OPERATOR_JUDGE_MAP.get(conditionData.getOperator()).judge(conditionData, realData);
    }
}
```

&ensp;&ensp;&ensp;&ensp;&ensp;一层又一层，继续跟进match函数中

```java
public class MatchOperatorJudge implements OperatorJudge {

    @Override
    public Boolean judge(final ConditionData conditionData, final String realData) {
        if (Objects.equals(ParamTypeEnum.URI.getName(), conditionData.getParamType())) {
            return PathMatchUtils.match(conditionData.getParamValue().trim(), realData);
        }
        return realData.contains(conditionData.getParamValue().trim());
    }
}
```

&ensp;&ensp;&ensp;&ensp;&ensp;在这终于看到了具体的逻辑实现了，大致可以看出这是个字符串匹配

```java
public class PathMatchUtils {

    private static final AntPathMatcher MATCHER = new AntPathMatcher();

    public static boolean match(final String matchUrls, final String path) {
        // matchUrls = /http/** , path = /http/order/findById
        return Splitter.on(",").omitEmptyStrings().trimResults().splitToList(matchUrls).stream().anyMatch(url -> reg(url, path));
    }

}
```

&ensp;&ensp;&ensp;&ensp;&ensp;选择器的匹配大致就是这些，可以但到进行匹配，其中的过程还挺复杂的，隐约能感受到一点设计的思想，有点逐步拆分的感觉。这块具体的分析，后面有时间再看看

&ensp;&ensp;&ensp;&ensp;&ensp;选择器匹配上以后，就进行到规则的匹配了，规则的匹配和选择器的匹配都是使用的这个匹配策略类进行匹配的，就是换行匹配的字符串罢了，这里就不详述了

&ensp;&ensp;&ensp;&ensp;&ensp;需要注意的一点是，规则匹配是使用请求的完整路径和规则的完整路径进行匹配的，没有截取之类的，也就是选择器和规则的路径设置存在高度的关联性，前缀可以说必须进行继承，这样感觉可能会导致一些灵活性的丧失

&ensp;&ensp;&ensp;&ensp;&ensp;继续来看 DividePlugin 插件，在下面的注释中可以看到 domain + readUrl 组合成了针对后端服务请求的url

```java
public class DividePlugin extends AbstractSoulPlugin {

    @Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        final DivideRuleHandle ruleHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), DivideRuleHandle.class);
        final List<DivideUpstream> upstreamList = UpstreamCacheManager.getInstance().findUpstreamListBySelectorId(selector.getId());
        if (CollectionUtils.isEmpty(upstreamList)) {
            log.error("divide upstream configuration error： {}", rule.toString());
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        final String ip = Objects.requireNonNull(exchange.getRequest().getRemoteAddress()).getAddress().getHostAddress();
        DivideUpstream divideUpstream = LoadBalanceUtils.selector(upstreamList, ruleHandle.getLoadBalance(), ip);
        if (Objects.isNull(divideUpstream)) {
            log.error("divide has no upstream");
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        // set the http url : http://192.168.101.104:8188
        String domain = buildDomain(divideUpstream);
        // get real url : http://192.168.101.104:8188/order/findById?id=1111
        String realURL = buildRealURL(domain, soulContext, exchange);
        exchange.getAttributes().put(Constants.HTTP_URL, realURL);
        exchange.getAttributes().put(Constants.HTTP_TIME_OUT, ruleHandle.getTimeout());
        exchange.getAttributes().put(Constants.HTTP_RETRY, ruleHandle.getRetry());
        return chain.execute(exchange);
    }
}
```

&ensp;&ensp;&ensp;&ensp;&ensp;不知道是不是平时使用的是NGINX，所有感觉Soul网关的转发好像不是那么灵活

&ensp;&ensp;&ensp;&ensp;&ensp;比如我们配置： http://test/baidu ，转发到百度后端服务器： http://baidu.com

&ensp;&ensp;&ensp;&ensp;&ensp;如果我们按照两级来配置的话，那真实的url就会变成 http://baidu.com/baidu

&ensp;&ensp;&ensp;&ensp;&ensp;使用一级前缀配置能达到目的，都使用match，选择器配置一级前缀，规则配置 /** ，这样前缀为 test 的请求都会转到百度

&ensp;&ensp;&ensp;&ensp;&ensp;上面转发成功还是因为 规则： /** 能匹配 /test/ ，感觉没有NGINX类似的截取之类

## 总结
&ensp;&ensp;&ensp;&ensp;&ensp;通过分析Soul的匹配算法，对如果写配置有了更深的了解，下面两点是需要注意的：

&ensp;&ensp;&ensp;&ensp;&ensp;1.Soul网关只支持一级前缀，因为在设置RealURL的时候，分隔字符串后定时取取str[0]为前缀

&ensp;&ensp;&ensp;&ensp;&ensp;2.Soul网关选择器和规则的路径设置存在高度的关联性，前缀可以说必须进行继承

&ensp;&ensp;&ensp;&ensp;&ensp;了解了匹配的一些细节，有助于写匹配

## Soul网关源码分析文章列表
### Github
- [Soul源码阅读（一） 概览](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB1-%E6%A6%82%E8%A7%88.md)
- [Soul源码阅读（二）代码初步运行](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB2-%E5%88%9D%E6%AD%A5%E8%BF%90%E8%A1%8C.md)
- [Soul源码阅读（三）HTTP请求处理概览](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB3-%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%A6%82%E8%A7%88.md)
- [Soul网关源码阅读（四）Dubbo请求概览](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB4-dubbo%E8%AF%B7%E6%B1%82%E6%A6%82%E8%A7%88.md)
- [Soul网关源码阅读（五）请求类型探索](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB5-%E8%AF%B7%E6%B1%82%E7%B1%BB%E5%9E%8B%E6%8E%A2%E7%B4%A2.md)
- [Soul网关源码阅读（六）Sofa请求处理概览](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB6-sofa%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%A6%82%E8%A7%88.md)
- [Soul网关源码阅读（七）限流插件初探](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB7-%E9%99%90%E6%B5%81%E6%8F%92%E4%BB%B6%E5%88%9D%E6%8E%A2.md)

- [Soul网关源码阅读番外篇（一） HTTP参数请求错误](https://github.com/lw1243925457/SE-Notes/blob/master/profession/program/%E5%BC%80%E6%BA%90/soul/soul%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%95%AA%E5%A4%96%E7%AF%871-HTTP%E7%A4%BA%E4%BE%8B%E5%8F%82%E6%95%B0%E8%AF%B7%E6%B1%82%E9%94%99%E8%AF%AF.md)

### 掘金
- [Soul网关源码阅读（一） 概览](https://juejin.cn/post/6917864624423436296)
- [Soul网关源码阅读（二）代码初步运行](https://juejin.cn/post/6917865804121767944)
- [Soul网关源码阅读（三）请求处理概览](https://juejin.cn/post/6917866538712334343)
- [Soul网关源码阅读（四）Dubbo请求概览](https://juejin.cn/post/6917867369909977102)
- [Soul网关源码阅读（五）请求类型探索](https://juejin.cn/post/6918575905962983438)
- [Soul网关源码阅读（六）Sofa请求处理概览](https://juejin.cn/post/6918736260467015693)
- [Soul网关源码阅读（七）限流插件初探](https://juejin.cn/post/6919348164944232455/)

- [Soul网关源码阅读番外篇（一） HTTP参数请求错误](https://juejin.cn/post/6918947689564471309)