# Webflux 中得到请求内容
***
## 代码示例
```java
public class PutBodyPlugin implements SoulPlugin {

    @Override
    public Mono<Void> execute(ServerWebExchange exchange, SoulPluginChain chain) {
        final ServerHttpRequest request = exchange.getRequest();
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        request.getBody().doOnNext(dataBuffer -> {
            try {
                Channels.newChannel(baos).write(dataBuffer.asByteBuffer().asReadOnlyBuffer());
            } catch (IOException e) {
                e.printStackTrace();
            }
            String body = new String(baos.toByteArray(), StandardCharsets.UTF_8);
            System.out.println(body);
        }).subscribe();
        return chain.execute(exchange);
    }

    @Override
    public int getOrder() {
        return 0;
    }
}
```

## 参考链接
- [How to log request body in spring Webflux Java](https://stackoverflow.com/questions/61706948/how-to-log-request-body-in-spring-webflux-java)