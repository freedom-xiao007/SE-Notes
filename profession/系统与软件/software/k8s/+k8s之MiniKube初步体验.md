# K8s 之 Minikube 初步体验
***

这是我参与2022首次更文挑战的第21天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7052884569032392740)

## 简介
目前想学习和了解云原生相关的东西，首选就是K8s，在官网上介绍Minikube是一个单体简单的，可以用于快速入门的东西，由于在本篇文章中探索下



## 安装运行

本人是Win11系统，下面的讲解也都是基于Win11的,记得安装相应的Docker环境

在下面的链接中，有相关系统的下载地址，下载后按照即可：

- [minikube start](https://minikube.sigs.k8s.io/docs/start/)

由于懒，没有设置环境变量，直接进入安装目录，运行命令：

```sh
cd D:\SoftWare\Kubernetes\Minikube\
.\minikube.exe start
```

需要等待一段时间，需求去下载和拉取镜像，完成后输出如下，不得不说，哪些表情还挺花里胡哨的，后面打算研究下，这个是怎么搞出来的！

```text
D:\SoftWare\Kubernetes\Minikube> .\minikube.exe start
😄  Microsoft Windows 11 Home China 10.0.22000 Build 22000 上的 minikube v1.25.1
✨  自动选择 docker 驱动
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
💾  Downloading Kubernetes v1.23.1 preload ...
    > preloaded-images-k8s-v16-v1...: 504.42 MiB / 504.42 MiB  100.00% 8.40 MiB
    > index.docker.io/kicbase/sta...: 378.98 MiB / 378.98 MiB  100.00% 1.76 MiB
❗  minikube was unable to download gcr.io/k8s-minikube/kicbase:v0.0.29, but successfully downloaded docker.io/kicbase/stable:v0.0.29 as a fallback image
🔥  Creating docker container (CPUs=2, Memory=8000MB) ...
❗  This container is having trouble accessing https://k8s.gcr.io
💡  To pull new external images, you may need to configure a proxy: https://minikube.sigs.k8s.io/docs/reference/networking/proxy/
🐳  正在 Docker 20.10.12 中准备 Kubernetes v1.23.1…
    ▪ kubelet.housekeeping-interval=5m
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

跟着教程命令输出一下，看看已经内置很多的东西

关键的一点是，在系统中使用docker ps命令，并没有看到这些容器，感觉不是用docker去创建的，有点像直接用RunC之类搞的，没有关联到Docker，前几天补了点知识，后面研究下

```sh
D:\SoftWare\Kubernetes\Minikube> kubectl get po -A
NAMESPACE     NAME                               READY   STATUS    RESTARTS      AGE
kube-system   coredns-64897985d-kk2ct            1/1     Running   0             31s
kube-system   etcd-minikube                      1/1     Running   0             42s
kube-system   kube-apiserver-minikube            1/1     Running   0             45s
kube-system   kube-controller-manager-minikube   1/1     Running   0             42s
kube-system   kube-proxy-x2lp5                   1/1     Running   0             31s
kube-system   kube-scheduler-minikube            1/1     Running   0             42s
kube-system   storage-provisioner                1/1     Running   1 (24s ago)   39s
```

然后是开启网页管理

```sh
D:\SoftWare\Kubernetes\Minikube> .\minikube.exe dashboard
🔌  正在开启 dashboard ...
    ▪ Using image kubernetesui/dashboard:v2.3.1
    ▪ Using image kubernetesui/metrics-scraper:v1.0.7
🤔  正在验证 dashboard 运行情况 ...
🚀  Launching proxy ...
🤔  正在验证 proxy 运行状况 ...
🎉  Opening http://127.0.0.1:64442/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/ in your default browser...
```

界面第一感觉还不错

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c67d48df59d64b6c9e2b330ce8cc25fb~tplv-k3u1fbpfcp-watermark.image?)

## 部署与运行应用
接下来我们来试着部署运行一个应用，教程中的示例运行不动，好像是网络访问有问题，我自己上传了一个简单的go的HTTP服务镜像，用于尝试，你们也可以直接pull下来，应用已经上传到docker hub中了

下面的命令是：

- 部署运行应用：名称 http-example-go 和对应的镜像
- 应用暴露的服务端口：8080
- 类似docker 的 -p，映射端口到宿主机

```sh
kubectl.exe create deployment http-example-go --image=lw1243925457/http_example:v1
kubectl.exe expose deployment http-example-go --type=NodePort --port=8080

kubectl.exe port-forward service/http-example-go 8080:8080

Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
```

然后我们就可以访问了: http://127.0.0.1:8080/v1/hello

## 总结
本篇介绍了Minicube的初步体验使用，目前还没有感受到K8s的强大，得继续研究和学习

后面尝试部署K8s集群，看看和单体有什么区别，部署应用之类的方不方便

## 参考链接
- [minikube start](https://minikube.sigs.k8s.io/docs/start/)
- [Hello Minikube](https://kubernetes.io/docs/tutorials/hello-minikube/)
- [Interactive Tutorial - Deploying an App](https://kubernetes.io/docs/tutorials/kubernetes-basics/deploy-app/deploy-interactive/)