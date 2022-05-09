# K8s V1.24版本 入门安装运行

***



```sh
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system

yum install -y nc

[root@VM-16-14-centos ~]# nc 127.0.0.1 6443
Ncat: Connection refused.
```



```sh
默认情况下，Kubernetes 使用 容器运行时接口（Container Runtime Interface，CRI） 来与你所选择的容器运行时交互。

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

setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

yum -y install kubelet kubeadm kubectl
systemctl enable --now kubelet


[root@VM-16-14-centos ~]# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"24", GitVersion:"v1.24.0", GitCommit:"4ce5a8954017644c5420bae81d72b09b735c21f0", GitTreeState:"clean", BuildDate:"2022-05-03T13:44:24Z", GoVersion:"go1.18.1", Compiler:"gc", Platform:"linux/amd64"}

[root@VM-16-14-centos ~]# kubectl version
WARNING: This version information is deprecated and will be replaced with the output from kubectl version --short.  Use --output=yaml|json to get the full version.
Client Version: version.Info{Major:"1", Minor:"24", GitVersion:"v1.24.0", GitCommit:"4ce5a8954017644c5420bae81d72b09b735c21f0", GitTreeState:"clean", BuildDate:"2022-05-03T13:46:05Z", GoVersion:"go1.18.1", Compiler:"gc", Platform:"linux/amd64"}
Kustomize Version: v4.5.4
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```



```sh
cat <<EOF | tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# 配置 sysctl 参数，这些配置在重启之后仍然起作用
cat <<EOF | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system
```



```sh
[root@VM-16-14-centos ~]# cat /etc/redhat-release
CentOS Linux release 7.5.1804 (Core)

在下列操作系统上安装 CRI-O, 使用下表中合适的值设置环境变量 OS:

操作系统	$OS
Centos 8	CentOS_8
Centos 8 Stream	CentOS_8_Stream
Centos 7	CentOS_7

然后，将 $VERSION 设置为与你的 Kubernetes 相匹配的 CRI-O 版本。 例如，如果你要安装 CRI-O 1.20, 请设置 VERSION=1.20. 你也可以安装一个特定的发行版本。 例如要安装 1.20.0 版本，设置 VERSION=1.20:1.20.0.
```



```sh
sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo
sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo


curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/CentOS_7/devel:kubic:libcontainers:stable.repo
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:1.24.0.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:1.24.0/CentOS_7/devel:kubic:libcontainers:stable:cri-o:1.24.0.repo

curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:1.23.0.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/1.23:/1.23.0/CentOS_7/devel:kubic:libcontainers:stable:cri-o:1.23:1.23.0.repo

yum install -y cri-o
systemctl daemon-reload
systemctl enable crio --now

sed -i "s/pause_image = .*/pause_image = \"registry.cn-hangzhou.aliyuncs.com\/google_containers\/pause:3.1\"/g" /etc/crio/crio.conf
systemctl restart crio

[root@VM-16-14-centos ~]# curl -v --unix-socket /var/run/crio/crio.sock http://localhost/info
* About to connect() to localhost port 80 (#0)
*   Trying /var/run/crio/crio.sock...
* Failed to set TCP_KEEPIDLE on fd 3
* Failed to set TCP_KEEPINTVL on fd 3
* Connected to localhost (/var/run/crio/crio.sock) port 80 (#0)
> GET /info HTTP/1.1
> User-Agent: curl/7.29.0
> Host: localhost
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: application/json
< Date: Sun, 08 May 2022 02:49:37 GMT
< Content-Length: 239
<
* Connection #0 to host localhost left intact
{"storage_driver":"overlay","storage_root":"/var/lib/containers/storage","cgroup_driver":"systemd","default_id_mappings":{"uids":[{"container_id":0,"host_id":0,"size":4294967295}],"gids":[{"container_id":0,"host_id":0,"size":4294967295}]}}
```



```sh
[root@VM-16-14-centos ~]# cat /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS=

[root@VM-16-14-centos ~]# echo "KUBELET_CGROUP_ARGS=--cgroup-driver=systemd" >> /etc/sysconfig/kubelet

[root@VM-16-14-centos ~]# cat /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS=
KUBELET_CGROUP_ARGS=--cgroup-driver=systemd

# !!在 kubelet 启动文件 /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf 增加 KUBELET_CGROUP_ARGS 参数 !!
# ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS $KUBELET_CGROUP_ARGS

# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS


systemctl daemon-reload && systemctl enable kubelet --now
systemctl restart kubelet

[root@VM-16-14-centos ~]# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: activating (auto-restart) (Result: exit-code) since Sun 2022-05-08 11:07:55 CST; 9s ago
     Docs: https://kubernetes.io/docs/
  Process: 10236 ExecStart=/usr/bin/kubelet (code=exited, status=1/FAILURE)
 Main PID: 10236 (code=exited, status=1/FAILURE)


[root@VM-16-14-centos ~]# kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.24.0
k8s.gcr.io/kube-controller-manager:v1.24.0
k8s.gcr.io/kube-scheduler:v1.24.0
k8s.gcr.io/kube-proxy:v1.24.0
k8s.gcr.io/pause:3.7
k8s.gcr.io/etcd:3.5.3-0
k8s.gcr.io/coredns/coredns:v1.8.6


[root@VM-16-14-centos k8s]# cat kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 1.2.3.4
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: node
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: 1.24.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}


apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 172.17.16.14
  bindPort: 6443
nodeRegistration:
  criSocket: "unix:///var/run/crio/crio.sock"
  taints:
  - effect: PreferNoSchedule
    key: node-role.kubernetes.io/master
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.24.0
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
networking:
  podSubnet: 10.85.0.0/16
  
  
[root@VM-16-14-centos k8s]# kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.24.0
k8s.gcr.io/kube-controller-manager:v1.24.0
k8s.gcr.io/kube-scheduler:v1.24.0
k8s.gcr.io/kube-proxy:v1.24.0
k8s.gcr.io/pause:3.7
k8s.gcr.io/etcd:3.5.3-0
k8s.gcr.io/coredns/coredns:v1.8.6

  
[root@VM-16-14-centos k8s]# kubeadm config images list --config kubeadm-config.yaml
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.24.0
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.24.0
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.24.0
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.24.0
registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.7
registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.3-0
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.6


vim /etc/crio/crio.conf
# 自己的配置  修改为自己的镜像仓库
insecure_registries = ["docker.io"]
pause_image = "docker.io/ningan123/pause:3.5"

systemctl daemon-reload
systemctl restart crio
systemctl status crio


[root@VM-16-14-centos k8s]# kubeadm init --config kubeadm-config.yaml
[init] Using Kubernetes version: v1.24.0
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
        [ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
[root@VM-16-14-centos k8s]# kubeadm reset



kubeadm reset
echo 1 > /proc/sys/net/ipv4/ip_forward
kubeadm init --config kubeadm-config.yaml



kubeadm init --apiserver-advertise-address=172.17.16.14 --pod-network-cidr=10.244.0.0/16 --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'
```



```sh
crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.24.0
crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.24.0
crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.24.0
crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.24.0
crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.7
crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.0-0
crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.6

[root@VM-16-14-centos k8s]# crictl images
IMAGE                                                                         TAG                 IMAGE ID            SIZE
docker.io/library/nginx                                                       latest              fa5269854a5e6       146MB
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns                   v1.8.6              a4ca41631cc7a       47MB
registry.cn-hangzhou.aliyuncs.com/google_containers/etcd                      3.5.0-0             0048118155842       296MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver            v1.24.0             529072250ccc6       131MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager   v1.24.0             88784fb4ac2f6       121MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy                v1.24.0             77b49675beae1       112MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler            v1.24.0             e3ed7dee73e93       52.3MB
registry.cn-hangzhou.aliyuncs.com/google_containers/pause                     3.7                 221177c6082a8       718kB

```

