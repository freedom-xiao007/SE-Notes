# Spring Valid 使用记录
***

## 基本使用
只需简单的在实体中添加注解和接口中添加注解即可：

```java
class User {
    
    @NotNull
    private String name;
}

class Controller {
    
    public String hello(@Validated @RequestParam("name") String name) {
        record "hello";
    }
}
```