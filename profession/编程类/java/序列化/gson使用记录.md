# Gson 使用记录
***

解决方法之一，定义自己的序列化

```java
ArrayList<HashMap<String, String>> arr = new ArrayList<>();
arr.add(new HashMap<String, String>() {{
    put("title", "123");
    put("link", "456");
}});
System.out.println(arr.toString());

Type type = new TypeToken<ArrayList<HashMap<String, String>>>() {}.getType();
System.out.println(new Gson().toJson(arr, type));
```

```java
logs = gson.fromJson(br, new TypeToken<List<JsonLog>>(){}.getType());
```

## 参考链接
- [How to read JSON array values in Spring controller](https://stackoverflow.com/questions/53327517/how-to-read-json-array-values-in-spring-controller)
- [Gson如何解决可变类型集合(Map/List)序列化](https://www.jianshu.com/p/fd647d3c2bf8)
- [Serialize ArrayList <HashMap> via GSON](https://stackoverflow.com/questions/61655920/serialize-arraylist-hashmap-via-gson)
- [Gson - convert from Json to a typed ArrayList<T>](https://stackoverflow.com/questions/12384064/gson-convert-from-json-to-a-typed-arraylistt)