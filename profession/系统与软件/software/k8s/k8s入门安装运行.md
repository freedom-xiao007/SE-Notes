# K8S入门-安装运行

***



```sh
➜  kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.24.0
k8s.gcr.io/kube-controller-manager:v1.24.0
k8s.gcr.io/kube-scheduler:v1.24.0
k8s.gcr.io/kube-proxy:v1.24.0
k8s.gcr.io/pause:3.7
k8s.gcr.io/etcd:3.5.3-0
k8s.gcr.io/coredns/coredns:v1.8.6
```



```sh
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.24.0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.24.0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.24.0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.24.0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.7
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.0-0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.6
```



```sh
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.24.0 k8s.gcr.io/kube-apiserver:v1.24.0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.24.0 k8s.gcr.io/kube-controller-manager:v1.24.0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.24.0 k8s.gcr.io/kube-scheduler:v1.24.0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.24.0 k8s.gcr.io/kube-proxy:v1.24.0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.7 k8s.gcr.io/pause:3.7
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.0-0 k8s.gcr.io/etcd:3.5.0-0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.6 k8s.gcr.io/coredns/coredns:v1.8.6
```



```sh
➜  ~ kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.24.0
k8s.gcr.io/kube-controller-manager:v1.24.0
k8s.gcr.io/kube-scheduler:v1.24.0
k8s.gcr.io/kube-proxy:v1.24.0
k8s.gcr.io/pause:3.7
k8s.gcr.io/etcd:3.5.3-0
k8s.gcr.io/coredns/coredns:v1.8.6
➜  ~ docker images |grep k8s
k8s.gcr.io/kube-apiserver                                                     v1.24.0   529072250ccc   3 days ago      130MB
k8s.gcr.io/kube-proxy                                                         v1.24.0   77b49675beae   3 days ago      110MB
k8s.gcr.io/kube-controller-manager                                            v1.24.0   88784fb4ac2f   3 days ago      119MB
k8s.gcr.io/kube-scheduler                                                     v1.24.0   e3ed7dee73e9   3 days ago      51MB
k8s.gcr.io/pause                                                              3.7       221177c6082a   8 weeks ago     711kB
k8s.gcr.io/coredns/coredns                                                    v1.8.6    a4ca41631cc7   7 months ago    46.8MB
k8s.gcr.io/etcd                                                               3.5.0-0   004811815584   10 months ago   295MB
```



```sh
kubeadm init --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers' --apiserver-advertise-address 0.0.0.0
kubeadm init --apiserver-advertise-address=10.0.20.11 --pod-network-cidr=10.244.0.0/16
kubeadm init --apiserver-advertise-address=10.0.20.11 --pod-network-cidr=10.244.0.0/16 --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'
kubeadm reset -f
```



## 参考链接

- [使用 kubeadm 快速部署 K8S V1.18](https://developer.aliyun.com/article/763983)
- [k8s-国内源安装](https://gist.github.com/islishude/231659cec0305ace090b933ce851994a)
- [[使用国内的镜像源搭建 kubernetes（k8s）集群](https://www.cnblogs.com/w84422/p/15596883.html)](https://www.cnblogs.com/w84422/p/15596883.html)

