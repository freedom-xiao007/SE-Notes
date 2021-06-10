# Spring Cloud Gateway 克隆配置
***
## 工程构建
&ensp;&ensp;&ensp;&ensp;下载工程到本地

```bash
# 先去github fork项目到自己的repo，然后克隆到自己本地
git clone --depth=1 https://github.com/lw1243925457/spring-cloud-gateway.git

# 切换
cd spring-cloud-gateway
git fetch origin v2.2.6.RELEASE:v2.2.6.RELEASE
git checkout v2.2.6.RELEASE
```

&ensp;&ensp;&ensp;&ensp;在依赖安装的时候（为了快速，使用的阿里云）可能会遇到依赖找不到的问题，如下面两个，解决方法指定 version，配置如下：

```bash
<groupId>org.codehaus.mojo</groupId>
<artifactId>flatten-maven-plugin</artifactId>
<version>1.2.5</version>

<artifactId>kotlin-maven-plugin</artifactId>
<groupId>org.jetbrains.kotlin</groupId>
<version>1.4.30-M1</version>
```

&ensp;&ensp;&ensp;&ensp;正常运行 spring-cloud-gateway-sample，正常访问网页：http://httpbin.org/