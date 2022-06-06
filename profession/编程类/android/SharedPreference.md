# SharedPreference 使用记录

***



遍历内容：

```java
private void getAllsp() {
    Map<String, ?> key_Value = (Map<String, ?>) sp.getAll(); //获取所有保存在对应标识下的数据，并以Map形式返回
 
    // 只需遍历即可得到存储的key和value值
 
    // 方法一
    for (Map.Entry<String, ?> entry : key_Value.entrySet()) {
        Log.i("获取的key：" + entry.getKey(), "获取的value:" + entry.getValue());
    }
    for (Map.Entry<String, ?> entry : key_Value.entrySet()) {
        System.out.println("key:" + entry.getKey() + " "
                + "value:" + entry.getValue());
    }
 
    // 方法二
    for (String key : key_Value.keySet()) {
        System.out.println("key:" + key + " " + "Value:" + key_Value.get(key));
    }
```





## 参考链接

- [遍历SharedPreferences中的数据，遍历map方法](https://blog.csdn.net/weixin_38107457/article/details/121962869)