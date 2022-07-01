# K8S V1.23 安装--Kubeadm+contained+公网 IP 多节点部署

---

## 简介

基于两台公网的服务器节点，两个服务器不再局域网内，只能通过公网 IP 相互访问，搭建 K8S 集群，并且按照 Dashboard，通过网页查看 K8S 相关的东西

## 环境及机器说明

两台机器，其中一台作为主节点，一台作为工作节点

操作系统都是centos7，centos8配置虚拟网卡有点麻烦

- crio-master（主节点）：121.4.190.84
- vm-20-11-centos(工作节点)：106.55.227.160

## 系统设置准备

同时在两台机器上执行

根据官方的文档，配置下一些系统属性

```sh
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system

# 修改Hosts文件，添加相关的配置，示例如下：
[root@crio-master k8s]# cat /etc/hosts
121.4.190.84 crio-master
106.55.227.160 VM-20-11-centos
```

由于我们使用的是公网 ip，但是在云服务器中是没有对应的网卡的，这导致在 kubeadm 部署时使用公网 IP 有问题

所以我们在两台机器中新建对应各自公网 ip 的虚拟网卡（下面方式建立的重启后，会被删除，但目前也够用了）

```shell
# 安装软件包
modprobe tun
lsmod | grep tun

# 编辑文件
vim /etc/yum.repos.d/nux-misc.repo
# 填入下面的内容
[nux-misc]
name=Nux Misc
baseurl=http://li.nux.ro/download/nux/misc/el7/x86_64/
enabled=0
gpgcheck=1
gpgkey=http://li.nux.ro/download/nux/RPM-GPG-KEY-nux.ro

# 安装
yum --enablerepo=nux-misc install tunctl

# 新建虚拟网卡
tunctl -t publick -u root
# 配置网卡的IP，注意替换ip成自己机器对应的公网IP
ifconfig publick 121.37.246.218 netmask 255.255.255.0 promisc
```

注：**K8S 部署需要开通 6443 端口，在服务器的安全规则配置中，将 6443 端口开启**

## containd 安装

两个机器上都安装

docker 作为我们日常经常使用的，但感觉比较重了，我们尝试不使用 docker，使用推荐的，较底层的 contained（cri-o 也行，目前尝试下来，没有想象中那么难用）

```shell
# 将机器人上的docker清理下，不然会有影响
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
yum remove docker-ce

# 我们单独安装contained即可
yum install -y yum-utils
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
yum install containerd.io

# 加载
systemctl daemon-reload

# 启动服务
systemctl enable containerd
systemctl start containerd
systemctl status containerd
```

## crictl 安装

两个机器上都安装

容器运行时的命令行操作工具，和 docker 命令很像，可以类比 docker，如下命令：

- docker ps == crictl ps
- docker logs == crictl logs

```shell
# 下载安装,下载不了，到github上找个版本：https://github.com/kubernetes-sigs/cri-tools/releases?q=v1.23.1&expanded=true，把下面的version改改就行了
VERSION="v1.24.1"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz

# 编辑配置文件
vim /etc/crictl.yaml
# 填入下面的内容
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 2
debug: false
pull-image-on-create: false
```

这样就 OK 了，可以运行命令尝试下：

```shell
crictl ps
crictl images
```

### 错误处理记录

###### FATA[0000] listing containers: rpc error: code = Unimplemented desc = unknown service runtime.v1alpha2.RuntimeService

```shell
➜  ~ crictl ps
FATA[0000] listing containers: rpc error: code = Unimplemented desc = unknown service runtime.v1alpha2.RuntimeService
```

删除下配置文件

```sh
rm /etc/containerd/config.toml
$ systemctl restart containerd
```

## k8s 安装

两个机器都需要执行

```shell
# 增加软件源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
 http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 设置下
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# 清理以前的版本，如果有的话
yum remove kubelet kubeadm kubectl
yum install -y kubelet-1.23.1 kubeadm-1.23.1 kubectl-1.23.1

# 启动
systemctl enable --now kubelet

# 编辑下配置文件
mkdir -p /etc/systemd/system/kubelet.service.d/
vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
# 填入下面的内容
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# 这是 "kubeadm init" 和 "kubeadm join" 运行时生成的文件，动态地填充 KUBELET_KUBEADM_ARGS 变量
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# 这是一个文件，用户在不得已下可以将其用作替代 kubelet args。
# 用户最好使用 .NodeRegistration.KubeletExtraArgs 对象在配置文件中替代。
# KUBELET_EXTRA_ARGS 应该从此文件中获取。
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS --node-ip=159.138.89.50

# 重新启动，应用配置
systemctl daemon-reload
systemctl restart kubelet
systemctl status kubelet

# 查看下状态
[root@VM-16-14-centos k8s]# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Sun 2022-05-08 14:00:11 CST; 54s ago
     Docs: https://kubernetes.io/docs/
 Main PID: 12618 (kubelet)
   CGroup: /system.slice/kubelet.service
           └─12618 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime=remot...

# 查看日志
journalctl -xeu kubelet


# 查看相关的版本
[root@VM-16-14-centos ~]# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.0", GitCommit:"ab69524f795c42094a6630298ff53f3c3ebab7f4", GitTreeState:"clean", BuildDate:"2021-12-07T18:15:11Z", GoVersion:"go1.17.3", Compiler:"gc", Platform:"linux/amd64"}

[root@VM-16-14-centos ~]# kubectl version
Client Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.0", GitCommit:"ab69524f795c42094a6630298ff53f3c3ebab7f4", GitTreeState:"clean", BuildDate:"2021-12-07T18:16:20Z", GoVersion:"go1.17.3", Compiler:"gc", Platform:"linux/amd64"}
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

由于在国内，不能得到谷歌的官方镜像（虽然后面我们使用的是国内镜像，但可能是有 bug，pause 这个镜像还是会到谷歌去拉取

我们需要手动拉取国内的镜像，然后改成谷歌的镜像,运行下面的命令即可：

```shell
crictl pull registry.aliyuncs.com/google_containers/pause:3.6
ctr i pull registry.aliyuncs.com/google_containers/pause:3.6
ctr -n k8s.io i tag registry.aliyuncs.com/google_containers/pause:3.6 k8s.gcr.io/pause:3.6
```

## 主节点启动

下面这些操作，只需要在主节点：crio-master 上执行即可

下面终于到了关键 Kubeadm 初始化安装 K8s 主节点

```shell
# --apiserver-advertise-address 填写公网ip
# --service-cidr --pod-network-cidr 照抄填写即可
# --image-repository 配置国内镜像源
[root@crio-master k8s]# kubeadm init --apiserver-advertise-address=121.4.190.84 --service-cidr=10.1.0.0/16 --pod-network-cidr=192.168.0.0/16 --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'

# 下面是安装中生成的日志
I0626 12:19:55.601776   10881 version.go:255] remote version is much newer: v1.24.2; falling back to: stable-1.23
[init] Using Kubernetes version: v1.23.8
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [crio-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.1.0.1 121.4.190.84]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [crio-master localhost] and IPs [121.4.190.84 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [crio-master localhost] and IPs [121.4.190.84 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 12.002365 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.23" in namespace kube-system with the configuration for the kubelets in the cluster
NOTE: The "kubelet-config-1.23" naming of the kubelet ConfigMap is deprecated. Once the UnversionedKubeletConfigMap feature gate graduates to Beta the default name will become just "kubelet-config". Kubeadm upgrade will handle this transition transparently.
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node crio-master as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node crio-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: m5yedd.clmq0lw4s4d961yu
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

# 这段日志很关键，工作节点加入是，复制下面这个运行即可
kubeadm join 121.4.190.84:6443 --token m5yedd.clmq0lw4s4d961yu \
        --discovery-token-ca-cert-hash sha256:7c3ac34b89c2c4a079dd5684286c6306abc8b5dd98fdd1ea8f1f1df8f254f256


# 运行下面的命令
[root@crio-master k8s]# mkdir -p $HOME/.kube
[root@crio-master k8s]#  cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
cp: overwrite ‘/root/.kube/config’?  chown $(id -u):$(id -g) $HOME/.kube/config
[root@crio-master k8s]# mkdir -p $HOME/.kube
[root@crio-master k8s]# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
cp: overwrite ‘/root/.kube/config’? y
[root@crio-master k8s]# chown $(id -u):$(id -g) $HOME/.kube/config
```

接着安装下面网络插件:Calico

```shell
# 运行下面的命令
kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
kubectl create -f https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml

# 查看部署状态
watch kubectl get pods -n calico-system
```

安装完成，我们可以运行下面的命令看看 pod 状态和看看容器运行状态：

```shell
[root@crio-master k8s]# kubectl get nodes
NAME              STATUS   ROLES                  AGE     VERSION
crio-master       Ready    control-plane,master   3d1h    v1.23.1

[root@crio-master k8s]# crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID
386c53701c331       6cba34c44d478       3 days ago          Running             calico-apiserver          0                   c96c43e126a15
e822cc35cc1c7       6cba34c44d478       3 days ago          Running             calico-apiserver          0                   b439d9c51353c
335493793963d       a4ca41631cc7a       3 days ago          Running             coredns                   0                   f8b273d0b213a
8e25629c36fe5       ec95788d0f725       3 days ago          Running             calico-kube-controllers   0                   706ceebf584f7
f8bbdbc140cdb       a4ca41631cc7a       3 days ago          Running             coredns                   0                   e6a73ececa384
5be790a6dd5a9       a3447b26d32c7       3 days ago          Running             calico-node               0                   1728a55642148
f77615c9ea6a8       22336acac6bba       3 days ago          Running             calico-typha              0                   dd612b2d122b8
a887f081de0f5       9735044632553       3 days ago          Running             tigera-operator           0                   e7d02d9eb77c5
df1d4d706ad2d       db4da8720bcb9       3 days ago          Running             kube-proxy                0                   4b7a385e851a7
142aedbdc00c1       09d62ad3189b4       3 days ago          Running             kube-apiserver            0                   29033f38ec856
e457ace9d5f03       25f8c7f3da61c       3 days ago          Running             etcd                      0                   b9307128a6142
28b0581787520       2b7c5a0399844       3 days ago          Running             kube-controller-manager   7                   fa0f2ccf4585c
fc4f7f3d1bb6f       afd180ec7435a       3 days ago          Running             kube-scheduler            7                   b7c2064e73acf
```

## 节点加入集群

主节点安装完成了，下面看着按照工作节点

主节点有相关的调度控制相关组件，比如上面的:kube-controller-manager, kube-scheduler

工作节点则没有太多的组件，所以工作节点

在 master 节点执行 kubeadm token list 获取 token（注意查看是否过期）

```sh
kubeadm token list

# 如果是过期了，需要重新生成
kubeadm token create --print-join-command
```

设置一下，然后运行命令让当前 worker 节点加入

```shell
modprobe br_netfilter
echo 1 > /proc/sys/net/ipv4/ip_forward

kubeadm join 121.4.190.84:6443 --token m5yedd.clmq0lw4s4d961yu \
        --discovery-token-ca-cert-hash sha256:7c3ac34b89c2c4a079dd5684286c6306abc8b5dd98fdd1ea8f1f1df8f254f256
```

我们回到主节点，查看状态，看到了完成：

```sh
[root@crio-master k8s]# kubectl get nodes
NAME              STATUS   ROLES                  AGE    VERSION
crio-master       Ready    control-plane,master   3d2h   v1.23.1
vm-20-11-centos   Ready    <none>                 3d1h   v1.23.1
```

###### 没有 Ready 的可能原因记录

1.hosts 需要配置，需要节点 hostname 和公网 ip 对应，主节点和工作节点的 hosts 需要配置好

```sh
[root@crio-master k8s]# cat /etc/hosts
121.4.190.84 crio-master
106.55.227.160 VM-20-11-centos
```

2.6443 端口需要放开

## Dashboard 安装访问

### 安装配置

在 master 节点机器上安装即可

```shell
# 运行后即部署
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml

# 查看状态，等待状态完毕
[root@crio-master k8s]# kubectl get pods --namespace=kubernetes-dashboard -o wide
NAME                                         READY   STATUS    RESTARTS   AGE   IP              NODE              NOMINATED NODE   READINESS GATES
dashboard-metrics-scraper-799d786dbf-prrgr   1/1     Running   0          12m   192.168.3.194   vm-20-11-centos   <none>           <none>
kubernetes-dashboard-546cbc58cd-c5rs9        1/1     Running   0          12m   192.168.3.193   vm-20-11-centos   <none>           <none>

# 查看详细信息
kubectl describe pod dashboard-metrics-scraper-799d786dbf-prrgr --namespace=kubernetes-dashboard
# 查看节点信息
[root@crio-master ~]# kubectl --namespace=kubernetes-dashboard get service kubernetes-dashboard
NAME                   TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes-dashboard   ClusterIP   10.1.45.51   <none>        443/TCP   5m24s

# 编辑，将最后的type：ClusterIP改为type：NodePort
kubectl --namespace=kubernetes-dashboard edit service kubernetes-dashboard
# 再次运行命令查看，得到其映射到主机的端口32625，这样既可通过注解ip+端口访问
[root@crio-master ~]# kubectl --namespace=kubernetes-dashboard get service kubernetes-dashboard
NAME                   TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.1.45.51   <none>        443:32652/TCP   7m31s
```

### token 生成

```shell
# 新建用户
vim admin-user.yaml
# 填入下面的内容
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
# 执行下面的命令
kubectl create -f admin-user.yaml

# 绑定用户关系
vim admin-user-role-binding.yaml
# 填入下面的内容
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
# 执行下面的命令
kubectl create -f admin-user-role-binding.yaml
```

如果过程中提示存在或者需要删除，只需要 kubectl delete -f 相应的 yaml 文件即可。

获取 token，执行下面的命令

```shell
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

### 访问页面

在上面的部署方式中，我们通过名称查看到服务部署在第二台主机上：vm-20-11-centos

```shell
[root@crio-master k8s]# kubectl get pods --namespace=kubernetes-dashboard -o wide
NAME                                         READY   STATUS    RESTARTS   AGE   IP              NODE              NOMINATED NODE   READINESS GATES
dashboard-metrics-scraper-799d786dbf-prrgr   1/1     Running   0          12m   192.168.3.194   vm-20-11-centos   <none>           <none>
kubernetes-dashboard-546cbc58cd-c5rs9        1/1     Running   0          12m   192.168.3.193   vm-20-11-centos   <none>           <none>
```

博主尝试通过正常的方式访问：通过 service 访问，但不行，经过尝试，咱只能直接访问容器了

首先登录 vm-20-11-centos 这台机器，我们看下运行中的容器：

```shell
➜  k8s crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                        ATTEMPT             POD ID
93c138b35118e       7801cfc6d5c07       29 hours ago        Running             dashboard-metrics-scraper   0                   87f8f420a8614
c1e3fc2d6c6d9       57446aa2002e1       29 hours ago        Running             kubernetes-dashboard        0                   68897036dab5f
db3684cf5e6a2       a3447b26d32c7       2 days ago          Running             calico-node                 0                   cb13bab18abe7
5811551b53ad0       db4da8720bcb9       2 days ago          Running             kube-proxy                  1                   2960526e1e594
```

看到 kubernetes-dashboard 这个容器，通过查询资料，这个服务监听在 8443 端口，在 kubectl get pods --namespace=kubernetes-dashboard -o wide 中看到其 ip 是 192.168.3.193，那访问的地址就是：192.168.3.193:8443

访问下，确实能通：

```shell
➜  k8s curl 192.168.3.193:8443
Client sent an HTTP request to an HTTPS server.
```

由于是在云服务上，开太多的端口不安全，我们通过 nginx 配置来访问，安装一个 nginx(安装教程网上一大推，这里就不赘述了)

我们编辑 nginx 配置文件：/etc/nginx/nginx.conf,填入下面的内容

```nginx
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;

    server {
        listen       443 ssl http2;
        listen       [::]:443 ssl http2;
        server_name  _;
        root         /usr/share/nginx/html;

        ssl_certificate "/etc/nginx/cert/selfgrowth.club_bundle.crt";
        ssl_certificate_key "/etc/nginx/cert/selfgrowth.club.key";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location /k8sadmin/ {
            proxy_pass https://192.168.3.193:8443/;
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
}
```

证书需要自己去随便搞一个了：

- 1.如果有自己的域名，在对应的服务商申请生成 ssl 证书即可
- 2.没有的话，可以使用 OpenSSL 生成：[使用 OpenSSL 生成自签名证书](https://www.ibm.com/docs/zh/api-connect/10.0.1.x?topic=overview-generating-self-signed-certificate-using-openssl)

只是为了让 nginx 能跑起来而已，实践访问不用那个域名，直接用 ip

下面我们就直接使用第二个节点的 ip 进行访问即可，token 填入之前生成获取到的 token


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fe7283ab8fc546c9bf28e7585045f4eb~tplv-k3u1fbpfcp-watermark.image?)

## 参考链接

- [docker ce install on centos](https://docs.docker.com/engine/install/centos/)
- [CentOs 8.1 安装 containerd](https://blog.csdn.net/tongzidane/article/details/114285188)
- [kubeadm init fails with node not found](https://github.com/kubernetes/kubeadm/issues/2370)
- [Getting started with containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)
- [使用 kubeadm 创建集群失败报 Unable to register node with API server](https://www.w3cjava.com/technical-articles/%E4%BA%91%E5%8E%9F%E7%94%9F/125058030.html)
- [Quickstart for Calico on Kubernetes](https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart)
- [部署和访问 Kubernetes 仪表板（Dashboard）](https://kubernetes.io/zh-cn/docs/tasks/access-application-cluster/web-ui-dashboard/)
- [Web 基础配置篇（十七）: Kubernetes dashboard 安装配置](https://cloud.tencent.com/developer/article/1634453)
- [circtl 发行安装包](https://github.com/kubernetes-sigs/cri-tools/releases?q=v1.23.1&expanded=true)
- [节点加入k8s集群如何获取token等参数值](https://blog.csdn.net/dazuiba008/article/details/94595451)
