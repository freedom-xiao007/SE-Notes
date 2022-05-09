# Java Spring Boot Open Feign 使用记录
***

### Get请求中，多个参数传递
推荐使用下面的方式：

```java
@FeignClient(name = "microservice-provider-user")
public interface UserFeignClient {
  @RequestMapping(value = "/get", method = RequestMethod.GET)
  public User get2(@RequestParam Map<String, Object> map);
}
```

## 参考链接
- [feign构造多参数请求](https://blog.csdn.net/m0_37556444/article/details/82561828)