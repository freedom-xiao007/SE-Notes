# Maven 使用记录
***

### 跳过单元测试

- DskipTests，不执行测试用例，但编译测试用例类生成相应的class文件至target/test-classes下。
- Dmaven.test.skip=true，不执行测试用例，也不编译测试用例类。

```shell
mvn package -Dmaven.test.skip=true  
```

## 参考链接
- [maven跳过单元测试-maven.test.skip和skipTests的区别](https://blog.csdn.net/arkblue/article/details/50974957)