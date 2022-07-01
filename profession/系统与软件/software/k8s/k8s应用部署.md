# K8S 应用部署
***

## 简介
在上篇文章中，我们安装了K8S的基础组件，并按照网页的Dashboard控制台，接下来我们尝试使用K8S来部署我们自己的应用

## 历史文章回顾

- [K8S V1.23 安装--Kubeadm+contained+公网 IP 多节点部署](https://juejin.cn/post/7114828680018264078)

## 总体概览
在上篇中我们安装好了K8S，并进行了访问，基于之前的基础，我们开始部署我们自己的应用

文章大致的主要内容如下：

- 1.搭建自己私有化的镜像仓库：使用公共的docker镜像也可以，博主想体验下私有仓库，用于K8S拉取镜像
- 2.使用Dashboard部署应用，并访问

### 私有镜像仓库搭建
注：这个搭建还是比较麻烦的，使用dockerhub的公共仓库也是没有问题，使用dockerhub的key跳过这部分

首先找台服务器，安装docker，安装这里就不说了，官方的教程很全面

- [官方安装教程](https://docs.docker.com/engine/install/centos/)

并且按照docker compose，运行下面的命令

```sh
yum install -y docker-compose-plugin
```

需要用到域名证书，这个和上篇一样：

- 1.如果有自己的域名，在对应的服务商申请生成 ssl 证书即可
- 2.没有的话，可以使用 OpenSSL 生成：[使用 OpenSSL 生成自签名证书](https://www.ibm.com/docs/zh/api-connect/10.0.1.x?topic=overview-generating-self-signed-certificate-using-openssl)

#### 生成HTTP认证文件

运行下面的命令，生成认证文件,username 和 password 换成你自己的用户和密码

```sh
mkdir /home/docker/auth

docker run --rm \
    --entrypoint htpasswd \
    httpd:alpine \
    -Bbn username password > /home/docker/auth/nginx.htpasswd
```

#### Registry 运行

编写docker-compose.yaml文件，使用文件配置部署比较好，直接用docker命令后序不好查看

```sh
# 先将文件夹用于存放docker register容器部署文件
mkdir registry
cd registry
# 编辑文件
vim docker-compose.yml

# 填入下面的内容
version: '3'

services:
  registry:
    image: registry
    ports:
      - "1443:443"
    environment:
      - REGISTRY_AUTH=htpasswd
      - REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm
      - REGISTRY_AUTH_HTPASSWD_PATH=/auth/nginx.htpasswd
      - REGISTRY_HTTP_ADDR=0.0.0.0:443
      - REGISTRY_HTTP_TLS_CERTIFICATE=/certs/xiuxian.plus_bundle.crt
      - REGISTRY_HTTP_TLS_KEY=/certs/xiuxian.plus.key
    volumes:
      - /var/lib/registry:/var/lib/registry
      - /home/docker/auth:/auth
      - /etc/nginx/cert:/certs
```

如上面的配置文件

我们先将相关的目录挂载下：

- /var/lib/registry:/var/lib/registry 挂载主机目录，存储下相关的镜像，避免重启后镜像丢失
- /home/docker/auth:/auth 将HTTP认证文件所在的目录挂载进去
- /etc/nginx/cert:/certs 将域名证书目录挂载进去

registry设置监听在443端口，并且将其映射到宿主机的1443端口

剩下的都是相关的配置，认证方式，所用的认证文件，相关的ssl文件等等

编写完成后，我们在docker-compose.yaml坐在的目录下运行下面的命令：

```sh
# 启动
docker compose up -d

# 看下运行状态
➜  auth docker ps
CONTAINER ID   IMAGE             COMMAND                  CREATED        STATUS        PORTS                                               NAMES
4322bed9e795   registry          "/entrypoint.sh /etc…"   23 hours ago   Up 22 hours   5000/tcp, 0.0.0.0:1443->443/tcp, :::1443->443/tcp   registry-registry-1
```

映射到1443端口是因为443端口被nginx占用了

如果你们的没有被占用，直接设置映射到443端口即可

博主的nginx相关配置如下：

```nginx
server {
    listen       443 ssl http2;
    listen       [::]:443 ssl http2;
    server_name  www.xiuxian.plus xiuxian.plus;
    root         /usr/share/nginx/html;

    ssl_certificate "/etc/nginx/cert/xiuxian.plus_bundle.crt";
    ssl_certificate_key "/etc/nginx/cert/xiuxian.plus.key";
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

     # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    # docker registry的访问都是v2开头的，我们设置v2的前缀都转到registry即可
    location /v2 {
        proxy_pass https://xiuxian.plus:1443/v2;
        client_max_body_size    500m;
    }

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}
```

#### docker 上传镜像
搭建完成后，我们可以直接使用docker登录到镜像仓库，如何上传镜像

```sh
# 登录
➜ docker login xiuxian.plus
Authenticating with existing credentials...
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded

# 上传镜像
docker push xiuxian.plus/auth-java:v2.0
```

这样基本上我们的私有镜像仓库已经可以了

### 使用Dashboard部署应用，并访问
#### 私有仓库访问秘钥生成
首先我们在k8s的主节点上创建用来访问我们私有仓库的秘钥：

运行下面的命令：

```sh
kubectl create secret docker-registry --namespace=xiuxian xiuxian.docker.register --docker-server=xiuxian.plus --docker-username=username --docker-password=password --docker-email=1243925457@qq.com
```

需要变化的配置如下：

- --namespace=xiuxian ：指定命名空间
- xiuxian.docker.register :秘钥名称
- -docker-server=xiuxian.plus ：镜像仓库地址
- --docker-username=username --docker-password=password --docker-email=1243925457@qq.com :用户名、密码和邮箱

创建好后备用，可以用下面的命令进行查看相关信息：

```sh
kubectl get secret --namespace=xiuxian xiuxian.docker.register --output="jsonpath={.data.\.dockerconfigjson}" | base64 --decode
kubectl get secret --namespace=xiuxian xiuxian.docker.register -o yaml
```

#### 应用部署
在上篇中，我们已经搭建好了K8S的Dashboard，我们便可以使用它来部署我们的应用

使用token登录Dashboard，如下：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a30ca361c3c74696ac10167f86a053aa~tplv-k3u1fbpfcp-watermark.image?)

token的获取，如果忘记了，可以在k8s的主节点上运行下面的命令进行获取

```sh
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

点击下面图中的，右上角的+号，我们新建一个deployment，选择表单创建，然后输入的内容大致如下：

![1656658689201.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cc9c679170a445209b2a2cb2f909ca7b~tplv-k3u1fbpfcp-watermark.image?)

命名空间和我们上面的对应上，如果有环境变量的话，可以在页面最后面填上，或者部署后进行修改也行

点击确定后，我们在左侧的deployment可以看到我们的应用：

![1656659240205.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7707d3c5ca8a466b9b96900cd4301706~tplv-k3u1fbpfcp-watermark.image?)

pods页面中可以看到具体的，显示部署在了那个节点：

![1656659293928.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/523ab624aff74e8699b491e8c4bb998d~tplv-k3u1fbpfcp-watermark.image?)

点击进去，可以看到更详细的，我们可以看到其分配到ip

![1656659400674.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d5a13f5dbc7b4a8799dc8472e0afa99a~tplv-k3u1fbpfcp-watermark.image?)

目前还不太熟悉k8s的访问方式（其实是博主service和其他方式没用尝试成功......），所以我们继续采用原始的访问方式去访问应用

#### 应用访问
我们登录应用部署的节点，查看下容器情况

```sh
# 查看当前运行的容器，可以看到我们的auth-java应用
➜  ~ crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                        ATTEMPT             POD ID
65f738a94ab43       7548f6570748a       6 hours ago         Running             auth-java                   0                   78d02b56fcfa6
93c138b35118e       7801cfc6d5c07       3 days ago          Running             dashboard-metrics-scraper   0                   87f8f420a8614
c1e3fc2d6c6d9       57446aa2002e1       3 days ago          Running             kubernetes-dashboard        0                   68897036dab5f
db3684cf5e6a2       a3447b26d32c7       5 days ago          Running             calico-node                 0                   cb13bab18abe7
5811551b53ad0       db4da8720bcb9       5 days ago          Running             kube-proxy                  1                   2960526e1e594

# 设置监听8080端口好像无效，查看下应用监听在那个端口，可以看大监听在9000端口
➜  ~ crictl exec 65f738a94ab43 netstat -apn
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 :::9000                 :::*                    LISTEN      1/java
tcp        0      0 ::ffff:192.168.3.206:38186 ::ffff:139.9.159.127:3306 ESTABLISHED 1/java
tcp        0      0 ::ffff:192.168.3.206:37896 ::ffff:139.9.159.127:3306 ESTABLISHED 1/java
tcp        0      0 ::ffff:192.168.3.206:38038 ::ffff:139.9.159.127:3306 ESTABLISHED 1/java
tcp        0      0 ::ffff:192.168.3.206:38090 ::ffff:139.9.159.127:3306 ESTABLISHED 1/java
tcp        0      0 ::ffff:192.168.3.206:38124 ::ffff:139.9.159.127:3306 ESTABLISHED 1/java
tcp        0      0 ::ffff:192.168.3.206:38218 ::ffff:139.9.159.127:3306 ESTABLISHED 1/java
tcp        0      0 ::ffff:192.168.3.206:37954 ::ffff:139.9.159.127:3306 ESTABLISHED 1/java
tcp        0      0 ::ffff:192.168.3.206:38076 ::ffff:139.9.159.127:3306 ESTABLISHED 1/java
tcp        0      0 ::ffff:192.168.3.206:38138 ::ffff:139.9.159.127:3306 ESTABLISHED 1/java
tcp        0      0 ::ffff:192.168.3.206:38198 ::ffff:139.9.159.127:3306 ESTABLISHED 1/java
Active UNIX domain sockets (servers and established)
Proto RefCnt Flags       Type       State         I-Node PID/Program name    Path
unix  2      [ ]         STREAM     CONNECTED     11525779 1/java
```

然后我们访问下：

```sh
➜  ~ curl http://192.168.3.206:9000/app/versionCheck\?version\=1
{"data":{"downloadUrl":null,"updateMsg":null,"latest":true},"code":200,"msg":null}# 
```

我们直接使用k8s分配的ip+容器应用监听的端口进行访问

如果看到之前博主写的：手动写docker系列的网络部分或者熟悉容器网络的，应该对这个比较熟悉

这样，我们基本上上就访问成功，Nice

## 总结
本篇文章中，我们基于上篇部署的环境，使用Dashboard部署了我们的应用，并且成功进行了访问

但方式不是很K8s，安装官方的说明，pods的IP会在重启后变化，需要一些更好的访问方式，这部分等博主后序尝试了

## 参考链接
- [Install Docker Compose CLI plugin](https://docs.docker.com/compose/install/compose-plugin/#installing-compose-on-linux-systems)
- [使用 kubectl 管理 Secret](https://kubernetes.io/zh-cn/docs/tasks/configmap-secret/managing-secret-using-kubectl/)
- [k8s 配置 Secret 集成Harbor](https://blog.csdn.net/qq_34285557/article/details/124473705)
- [从私有仓库拉取镜像](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/pull-image-private-registry/)
- [Copying Kubernetes Secrets Between Namespaces](https://www.revsys.com/tidbits/copying-kubernetes-secrets-between-namespaces/)
- [为容器设置环境变量](https://kubernetes.io/zh-cn/docs/tasks/inject-data-application/define-environment-variable-container/)
- [Docker Registry Server 搭建,配置免费HTTPS证书，及拥有权限认证、TLS 的私有仓库](https://cloud.tencent.com/developer/article/1041478)
- [私有仓库高级配置](https://yeasy.gitbook.io/docker_practice/repository/registry_auth)
- [docker 查询或获取私有仓库(registry)中的镜像](https://blog.csdn.net/hongweigg/article/details/78789638)