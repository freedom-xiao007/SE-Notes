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



## 构建

1.首先肯定是使用maven或者gradle进行构建，生成jar

2.然后是运行镜像构建命令：docker build -t app-java:v1 .

3.上面就生成了应用镜像，后面可以推送到云上仓库，用于拉取并部署



也可以结合Jenkins进行使用