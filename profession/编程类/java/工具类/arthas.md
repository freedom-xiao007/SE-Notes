# Arthas 使用日志
***
## 简介
简介请自行查看官方介绍：[Arthas（阿尔萨斯） 能为你做什么？](https://arthas.aliyun.com/doc/)

## 安装
安装官方教程：[https://arthas.aliyun.com/doc/install-detail.html](https://arthas.aliyun.com/doc/install-detail.html)

这里就不赘述了，Windows推荐全量安装，直接把相关的都给下下来

## 使用说明
推荐根据教程运行一遍使用示例：

- [快速入门](https://arthas.aliyun.com/doc/quick-start.html)
- [进阶使用](https://arthas.aliyun.com/doc/advanced-use.html)

### Windows使用注意事项
- 1.在运行示例时，Terminal最好使用管理员权限，不然会无法查看到相关Java进程
- 2.如果运行arthas-boot.jar，发现找不到其他的Java进程，可以使用jdk自带的jps命令，查到相应的pid，然后加到arthas-boot.jar运行命令后面，比如：java -jar arthas-boot.jar 1111
- 3.java -jar arthas-boot.jar运行时可能提升什么全路径，java的命令带全即可
- 4.profile的相关命令在windows上都不支持，感觉玩不下去了

## 参考链接
- [官方文档：https://arthas.aliyun.com/doc/quick-start.html](https://arthas.aliyun.com/doc/quick-start.html)