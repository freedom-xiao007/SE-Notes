# 简易Java Dockerfile编写

***

## 简介

本文将介绍一个通用的Java应用的Dockerfile模板，用于日常的Java应用的打包上传



## Dockfile

很简单，如下

```dockerfile
FROM openjdk:17-alpine
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories && apk update && apk add busybox-extras
ADD ./target/oauth-0.0.1-SNAPSHOT.jar /app/oauth-0.0.1-SNAPSHOT.jar
CMD java -jar /app/oauth-0.0.1-SNAPSHOT.jar
```

首先是基础镜像，这里选择17版本，尝尝鲜

然后我们修改了软件源，并按照了telnet，方便日后的网络相关的问题定位

第三是将当前目录的已经构建后的jar添加到容器中

最后是容器启动时就运行的命令，启动Java镜像