# Java Map
***
### 遍历
```java
# EntrySet and for Loop
public void iterateUsingEntrySet(Map<String, Integer> map) {
    for (Map.Entry<String, Integer> entry : map.entrySet()) {
        System.out.println(entry.getKey() + ":" + entry.getValue());
    }
}

# Iterator and EntrySet
public void iterateUsingIteratorAndEntry(Map<String, Integer> map) {
    Iterator<Map.Entry<String, Integer>> iterator = map.entrySet().iterator();
    while (iterator.hasNext()) {
        Map.Entry<String, Integer> entry = iterator.next();
        System.out.println(entry.getKey() + ":" + entry.getValue());
    }
}

# With Lambdas
public void iterateUsingLambda(Map<String, Integer> map) {
    map.forEach((k, v) -> System.out.println((k + ":" + v)));
}

# Stream API
public void iterateUsingStreamAPI(Map<String, Integer> map) {
    map.entrySet().stream()
      // ...
      .forEach(e -> System.out.println(e.getKey() + ":" + e.getValue()));
}
```

## 参考链接
- [Guava之Maps教程](https://blog.csdn.net/neweastsun/article/details/79944839)
- [你只会用 map.put？试试 Java 8 compute ，操作 Map 更轻松！](https://www.cnblogs.com/javastack/p/14537491.html)
- [Java 8 Map sort](https://blog.csdn.net/Hatsune_Miku_/article/details/73435479)
- [JavaSE入门——Map转换为List排序](https://blog.csdn.net/lfz9696/article/details/107894605)
- [Iterate over a Map in Java](https://www.baeldung.com/java-iterate-map)