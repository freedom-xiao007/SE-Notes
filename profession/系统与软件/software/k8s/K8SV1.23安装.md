# K8S V1.23 安装

***



### 系统设置准备

```sh
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system

modprobe tun
lsmod | grep tun
vim /etc/yum.repos.d/nux-misc.repo
[nux-misc]
name=Nux Misc
baseurl=http://li.nux.ro/download/nux/misc/el7/x86_64/
enabled=0
gpgcheck=1
gpgkey=http://li.nux.ro/download/nux/RPM-GPG-KEY-nux.ro

yum --enablerepo=nux-misc install tunctl

tunctl -t publick -u root
ifconfig publick 106.55.227.160 netmask 255.255.255.0 promisc
```



## containd安装



```shell
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
yum remove docker-ce
yum install -y yum-utils
yum install containerd.io

加载
systemctl daemon-reload 

启动服务
systemctl enable containerd
systemctl start containerd
systemctl status containerd
```



## crictl 安装



```shell
VERSION="v1.23.1"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz

vim /etc/crictl.yaml
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 2
debug: false
pull-image-on-create: false
```



### 错误处理记录

###### FATA[0000] listing containers: rpc error: code = Unimplemented desc = unknown service runtime.v1alpha2.RuntimeService

```shell
➜  ~ crictl ps
FATA[0000] listing containers: rpc error: code = Unimplemented desc = unknown service runtime.v1alpha2.RuntimeService
```



```sh
rm /etc/containerd/config.toml
$ systemctl restart containerd
```





## k8s 安装



```shell
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

清理以前的版本，如果有的话
yum remove kubelet kubeadm kubectl
yum install -y kubelet-1.23.1 kubeadm-1.23.1 kubectl-1.23.1
systemctl enable --now kubelet


vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
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
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS


systemctl daemon-reload
systemctl restart kubelet
systemctl status kubelet

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

journalctl -xeu kubelet




[root@VM-16-14-centos ~]# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.0", GitCommit:"ab69524f795c42094a6630298ff53f3c3ebab7f4", GitTreeState:"clean", BuildDate:"2021-12-07T18:15:11Z", GoVersion:"go1.17.3", Compiler:"gc", Platform:"linux/amd64"}

[root@VM-16-14-centos ~]# kubectl version
Client Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.0", GitCommit:"ab69524f795c42094a6630298ff53f3c3ebab7f4", GitTreeState:"clean", BuildDate:"2021-12-07T18:16:20Z", GoVersion:"go1.17.3", Compiler:"gc", Platform:"linux/amd64"}
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```



## 主节点启动



```shell
ctr i pull registry.aliyuncs.com/google_containers/pause:3.6
ctr i tag registry.aliyuncs.com/google_containers/pause:3.6 k8s.gcr.io/pause:3.6
 ctr -n k8s.io i tag registry.aliyuncs.com/google_containers/pause:3.6 k8s.gcr.io/pause:3.6
```





```shell
[root@crio-master k8s]# kubeadm init --apiserver-advertise-address=xxx.xxx.xxx.xxx --service-cidr=10.1.0.0/16 --pod-network-cidr=192.168.0.0/16 --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'

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
[certs] etcd/server serving cert is signed for DNS names [crio-master localhost] and IPs [xxx.xxx.xxx.xxx 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [crio-master localhost] and IPs [xxx.xxx.xxx.xxx 127.0.0.1 ::1]
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

kubeadm join xxx.xxx.xxx.xxx:6443 --token m5yedd.clmq0lw4s4d961yu \
        --discovery-token-ca-cert-hash sha256:7c3ac34b89c2c4a079dd5684286c6306abc8b5dd98fdd1ea8f1f1df8f254f256
        
        
[root@crio-master k8s]# mkdir -p $HOME/.kube
[root@crio-master k8s]#  cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
cp: overwrite ‘/root/.kube/config’?  chown $(id -u):$(id -g) $HOME/.kube/config
[root@crio-master k8s]# mkdir -p $HOME/.kube
[root@crio-master k8s]# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
cp: overwrite ‘/root/.kube/config’? y
[root@crio-master k8s]# chown $(id -u):$(id -g) $HOME/.kube/config
[root@crio-master k8s]#
```





## 节点加入集群



```shell
nginx -V|grep configure arguments: –with-http_ssl_module

kubectl get nodes
kubectl describe node vm-20-11-centos
```





```shell
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml

kubectl get pods --namespace=calico-system -o wide
```





## Dashboard安装访问

在master节点机器上安装即可

```shell
# 运行后即可
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





## 参考链接

- [docker ce install on centos](https://docs.docker.com/engine/install/centos/)

- [CentOs 8.1 安装containerd](https://blog.csdn.net/tongzidane/article/details/114285188)
- [kubeadm init fails with node not found](https://github.com/kubernetes/kubeadm/issues/2370)
- [Getting started with containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)
- [使用kubeadm创建集群失败报Unable to register node with API server](https://www.w3cjava.com/technical-articles/%E4%BA%91%E5%8E%9F%E7%94%9F/125058030.html)
- [Quickstart for Calico on Kubernetes](https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart)
- [部署和访问 Kubernetes 仪表板（Dashboard）](https://kubernetes.io/zh-cn/docs/tasks/access-application-cluster/web-ui-dashboard/)
- [Web基础配置篇（十七）: Kubernetes dashboard安装配置](https://cloud.tencent.com/developer/article/1634453)