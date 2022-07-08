# K8S探索之Service+Flannel本机及跨主机网络访问原理详解
***



## 简介

在上篇中，我们部署了我们的应用，但我们访问是直接在应用所在的容器，使用IP+Port的方式直接访问的，style不够k8s，本篇文章我们将使用service和跨主机访问



## 内容概览

目前我们的应用大致如下：

![k8s应用拓扑.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/822ba15a40564525b1c60327cbdc275b~tplv-k3u1fbpfcp-watermark.image?)

我们的应用目前是部署在工作节点1上面，如上篇所示，我们可以在工作节点1上面直接使用应用容器的ip+port进行访问

```sh
➜  ~ curl http://10.244.1.8:9000/app/versionCheck\?version\=1
{"data":{"downloadUrl":null,"updateMsg":null,"latest":true},"code":200,"msg":null}#
```



但上面有个问题，每次重启，我们的应用ip是会变化的，也就是不固定，这样对我们访问会造成麻烦，我们不可能每次重启都去替换我们的访问ip

针对这种情况，k8s提供的解决方案是：Service

Service有一个固定的集群内访问IP，能够感知其对应的pod的ip变化，自动对应上

Service解决了Pod重启后的IP变化问题，但还有一个问题是跨主机访问问题：pod重启后，不一定在原来的node节点上，这个时候对我们访问也会带来麻烦，需要访问节点是能跨节点访问

针对跨主机访问问题，目前有很多解决方案，比如：Flannel，本篇也是基于Flannel进行探索的

整个如下图所示：

![k8s跨主机拓扑.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9184d75062184311a0e9549b6cb90202~tplv-k3u1fbpfcp-watermark.image?)



在工作节点和主节点上，我们都能通过service的集群IP进行访问，如下：

```shell
➜  dashboard curl http://10.103.5.88:9000/app/versionCheck\?version\=1
{"data":{"downloadUrl":null,"updateMsg":null,"latest":true},"code":200,"msg":null}
```



目前的概况就如上所示，接下来的我们就开始探索其中的原理，主要是两方面：

- 1.在工作节点中，如何通过service访问到具体的应用
- 2.主节点上没有部署有应用，是如何通过service访问到具体应用的



## 准备工作

本篇文章基于下面的文章：

- [K8S V1.23 安装--Kubeadm+contained+公网 IP 多节点部署](https://juejin.cn/post/7114828680018264078)
- [K8S 应用部署](https://juejin.cn/post/7115303827980419085)



本篇中，将应用重新部署了下，将网络插件contained换成了flanned，具体参考文章：[[kubernetes] 跨云厂商使用公网IP搭建k8s v1.20.9集群](https://blog.51cto.com/xiaowangzai/5167661)

首先是参考链接配置eth0:1网卡

其他的也不复杂，我们先将之前部署的k8s卸载，然后初始化k8s集群即可



service的生成也比较简单，在Dashboard中部署的我们的应用时，设置service即可，如下图所示：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d45394c41df943f1b7f7ac0731b111c0~tplv-k3u1fbpfcp-watermark.image?)

部署应用的时候选择service为Internal，即只集群内访问，不绑定主机节点端口，端口就填我们的应用对应端口接口：第一个端口是service的监听端口，第二个端口是pod应用的监听端口



## 本机内service访问原理解析

我们在工作节点1-IP：106.55.227.160上，使用service的方式访问应用：

```sh
➜ curl http://10.103.5.88:9000/app/versionCheck\?version\=1
{"data":{"downloadUrl":null,"updateMsg":null,"latest":true},"code":200,"msg":null}
```



通过查阅资料，其原理是使用Iptables，具体如下：

我们首先看看10.103.5.88这个ip相关的iptables规则配置

```sh
➜  ~ iptables -S -t nat |grep 10.103.5.88
-A KUBE-SERVICES -d 10.103.5.88/32 -p tcp -m comment --comment "xiuxian/auth-java:tcp-9000-9000-c8kgf cluster IP" -m tcp --dport 9000 -j KUBE-SVC-ATUXI35NVHBYDGFQ
```

如上所示：-d 10.103.5.88/32 + --dport 9000 ：表示将访问10.103.5.88+9000端口的请求，转到链路规则KUBE-SVC-ATUXI35NVHBYDGFQ



所以我们接着看看KUBE-SVC-ATUXI35NVHBYDGFQ

```sh
➜  ~ iptables -S -t nat |grep KUBE-SVC-ATUXI35NVHBYDGFQ
-A KUBE-SVC-ATUXI35NVHBYDGFQ -m comment --comment "xiuxian/auth-java:tcp-9000-9000-c8kgf" -j KUBE-SEP-7ETKSBKARXRYVX3G
```

如上所示，这个又将这个请求转到了：KUBE-SEP-7ETKSBKARXRYVX3G



我们接着看KUBE-SEP-7ETKSBKARXRYVX3G

```sh
➜  ~ iptables -S -t nat |grep KUBE-SEP-7ETKSBKARXRYVX3G
-A KUBE-SEP-7ETKSBKARXRYVX3G -p tcp -m comment --comment "xiuxian/auth-java:tcp-9000-9000-c8kgf" -m tcp -j DNAT --to-destination 10.244.1.8:9000
```

如上所示，上面就行了目标地址转换：DNAT --to-destination 10.244.1.8:9000

将请求转到了我们的pod应用中



上面就是本机内应用访问的原理，就是通过配置iptables来达到目的，这些规则都是kube-proxy进行配置的

详细的细节可参考：《Kubernetes网络权威指南：基础、原理与实践》--4.1部分，这是本好书，网络出问题的时候，博主翻了这本书很多遍



## service跨主机访问原理解析

service本机访问是通过iptables，但如果是跨主机访问就麻烦了

在主节点IP：121.4.190.84上，使用命令，我们也是看到其配置了相同的iptables规则：

```sh
➜  ~ iptables -S -t nat |grep KUBE-SEP-7ETKSBKARXRYVX3G
-A KUBE-SEP-7ETKSBKARXRYVX3G -p tcp -m comment --comment "xiuxian/auth-java:tcp-9000-9000-c8kgf" -m tcp -j DNAT --to-destination 10.244.1.8:9000
```



也是去访问了10.244.1.8，但是，在主节点上，是没有这个pod的

并且由于这个ip：10.244.1.8是个内网ip，在不进行任何配置的情况下，在主节点主机上是没有版本进行访问的

针对这个问题，本文用的是flannel的vxlan方式进行解决（尝试过calico，配置有点复杂，有些解决模式在公网ip集群有点麻烦，经过不断的尝试，终于使用flannel成功了），下面是整个请求的链路图，后面进行详细的解释：



上图就是一个完整的请求及响应链路，下面我们一步一步的具体解释：



###### 1.service--10.103.5.88:9000 iptables转换

在主节点上，和工作节点一样的方式，通过命令查看iptables规则，我们能得到请求最终转发到了10.244.1.8:9000

```sh
-A KUBE-SERVICES -d 10.103.5.88/32 -p tcp -m comment --comment "xiuxian/auth-java:tcp-9000-9000-c8kgf cluster IP" -m tcp --dport 9000 -j KUBE-SVC-ATUXI35NVHBYDGFQ
-A KUBE-SVC-ATUXI35NVHBYDGFQ -m comment --comment "xiuxian/auth-java:tcp-9000-9000-c8kgf" -j KUBE-SEP-7ETKSBKARXRYVX3G
-A KUBE-SEP-7ETKSBKARXRYVX3G -p tcp -m comment --comment "xiuxian/auth-java:tcp-9000-9000-c8kgf" -m tcp -j DNAT --to-destination 10.244.1.8:9000
```



###### 2.请求路由到flannel网络接口

安装flannel网络插件后，就会生成其网络接口，用于跨主机的解包和封包,如下的flannel.1

```sh
➜  ifconfig
flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.244.0.0  netmask 255.255.255.255  broadcast 10.244.0.0
        inet6 fe80::f4f2:b0ff:fefa:7573  prefixlen 64  scopeid 0x20<link>
        ether f6:f2:b0:fa:75:73  txqueuelen 0  (Ethernet)
        RX packets 867  bytes 99948 (97.6 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1508  bytes 212521 (207.5 KiB)
        TX errors 0  dropped 8 overruns 0  carrier 0  collisions 0
```



使用命令查看路由有下面的规则：

```sh
➜  ip route
10.244.1.0/24 via 10.244.1.0 dev flannel.1 onlink
```

即将10.244.1.0/24的请求交给flannel.1处理

我们通过抓包看看：

```sh
➜  tcpdump -i flannel.1 -e -nn |grep 10.244.1.8
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on flannel.1, link-type EN10MB (Ethernet), capture size 262144 bytes
10:36:22.935837 f6:f2:b0:fa:75:73 > da:a9:48:cb:15:3b, ethertype IPv4 (0x0800), length 74: 10.244.0.0.8741 > 10.244.1.8.9000: Flags [S], seq 1220826812, win 29200, options [mss 1460,sackOK,TS val 2079738924 ecr 0,nop,wscale 7], length 0
10:36:22.968258 da:a9:48:cb:15:3b > f6:f2:b0:fa:75:73, ethertype IPv4 (0x0800), length 74: 10.244.1.8.9000 > 10.244.0.0.8741: Flags [S.], seq 4079516162, ack 1220826813, win 27960, options [mss 1410,sackOK,TS val 1027012415 ecr 2079738924,nop,wscale 7], length 0
10:36:22.968294 f6:f2:b0:fa:75:73 > da:a9:48:cb:15:3b, ethertype IPv4 (0x0800), length 66: 10.244.0.0.8741 > 10.244.1.8.9000: Flags [.], ack 1, win 229, options [nop,nop,TS val 2079738957 ecr 1027012415], length 0
10:36:22.968387 f6:f2:b0:fa:75:73 > da:a9:48:cb:15:3b, ethertype IPv4 (0x0800), length 172: 10.244.0.0.8741 > 10.244.1.8.9000: Flags [P.], seq 1:107, ack 1, win 229, options [nop,nop,TS val 2079738957 ecr 1027012415], length 106
10:36:23.002585 da:a9:48:cb:15:3b > f6:f2:b0:fa:75:73, ethertype IPv4 (0x0800), length 66: 10.244.1.8.9000 > 10.244.0.0.8741: Flags [.], ack 107, win 219, options [nop,nop,TS val 1027012447 ecr 2079738957], length 0
10:36:23.059860 da:a9:48:cb:15:3b > f6:f2:b0:fa:75:73, ethertype IPv4 (0x0800), length 268: 10.244.1.8.9000 > 10.244.0.0.8741: Flags [P.], seq 1:203, ack 107, win 219, options [nop,nop,TS val 1027012505 ecr 2079738957], length 202
10:36:23.059897 f6:f2:b0:fa:75:73 > da:a9:48:cb:15:3b, ethertype IPv4 (0x0800), length 66: 10.244.0.0.8741 > 10.244.1.8.9000: Flags [.], ack 203, win 237, options [nop,nop,TS val 2079739048 ecr 1027012505], length 0
10:36:23.060196 da:a9:48:cb:15:3b > f6:f2:b0:fa:75:73, ethertype IPv4 (0x0800), length 71: 10.244.1.8.9000 > 10.244.0.0.8741: Flags [P.], seq 203:208, ack 107, win 219, options [nop,nop,TS val 1027012505 ecr 2079738957], length 5
10:36:23.060206 f6:f2:b0:fa:75:73 > da:a9:48:cb:15:3b, ethertype IPv4 (0x0800), length 66: 10.244.0.0.8741 > 10.244.1.8.9000: Flags [.], ack 208, win 237, options [nop,nop,TS val 2079739049 ecr 1027012505], length 0
10:36:23.060283 f6:f2:b0:fa:75:73 > da:a9:48:cb:15:3b, ethertype IPv4 (0x0800), length 66: 10.244.0.0.8741 > 10.244.1.8.9000: Flags [F.], seq 107, ack 208, win 237, options [nop,nop,TS val 2079739049 ecr 1027012505], length 0
10:36:23.094810 da:a9:48:cb:15:3b > f6:f2:b0:fa:75:73, ethertype IPv4 (0x0800), length 66: 10.244.1.8.9000 > 10.244.0.0.8741: Flags [F.], seq 208, ack 108, win 219, options [nop,nop,TS val 1027012540 ecr 2079739049], length 0
^C24 packets captured
24 packets received by filter
0 packets dropped by kernel
```

关键点是这个：10.244.0.0.8741 > 10.244.1.8.9000:

在上面的将请求路由到flannel.1后，flannel进行封包处理

10.244.0.0 是flannel.1的ip地址，如上面使用ifconfig命令看到的

10.244.1.8.9000就是目标ip了



###### 3，4.封包后，请求发送到工作节点

我们监听下物理网卡eth0

```sh
➜  tcpdump -i eth0 -e -nn |grep -E '10.244.1.8|106.55.227.160.8472'
10:49:31.632150 52:54:00:f4:44:15 > fe:ee:b5:81:6a:26, ethertype IPv4 (0x0800), length 116: 172.17.16.14.51945 > 106.55.227.160.8472: OTV, flags [I] (0x08), overlay 0, instance 1
```

在我们发起情况后，看到出现上面的包：172.17.16.14.51945 > 106.55.227.160.8472

请求由我们的物理网卡eth0(ip就是172.17.16.14)，发送到工作节点（公网ip：106.55.227.160）的8472端口，8472端口是flannel预定的接口

从上面可以看到，如果出现网络不通的情况，检查下8741端口，我们需要开发主机的8741端口

从上面eth0我们没有看到我们的目标地址10.244.1.8的痕迹，那是怎么对应起来的，不急，我们接着往下看



###### 5,6flannel应用受到请求包，解包后，请求pod

看有的资料是配置的端口转发到flannel，但没找到，更像是flannel的应用监听在8472端口，具体方式也不像监听，为了便于理解，暂时视为监听在8472端口吧，如下：

```sh
# 每个node节点都有kube-flannel  
➜  ~ crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                        ATTEMPT             POD ID
24bfdfc099e0a       8522d622299ca       23 hours ago        Running             kube-flannel                0                   4bd04cec2eeb2
# 下面是相关的进程和端口情况，虽然PID为-，但在iptables没有找到转发规则，暂时看成flannel应用应该是监听在8472端口
➜  ~ ps aux|grep flannel
root     22561  0.0  0.5 1269468 23136 ?       Ssl  Jul07   0:47 /opt/bin/flanneld --ip-masq --kube-subnet-mgr --public-ip=106.55.227.160 --iface=eth0
➜  ~ netstat -ulnp|grep 8472
udp        0      0 0.0.0.0:8472            0.0.0.0:*                           -
```



抓下工作节点物理网卡的包，我们只抓8472端口的接口，因为上面也看到了主节点物理网卡是发到了工作节点的8472端口上的

```sh
# eth0物理网卡的
➜  ~ tcpdump -i eth0 -e -nn port 8472
11:20:29.533872 fe:ee:b8:5a:5a:63 > 52:54:00:4f:72:ca, ethertype IPv4 (0x0800), length 124: 121.4.190.84.59814 > 10.0.20.11.8472: OTV, flags [I] (0x08), overlay 0, instance 1
f6:f2:b0:fa:75:73 > da:a9:48:cb:15:3b, ethertype IPv4 (0x0800), length 74: 10.244.0.0.14571 > 10.244.1.8.9000: Flags [S], seq 2501773392, win 29200, options [mss 1460,sackOK,TS val 2082385506 ecr 0,nop,wscale 7], length 0

# flannel的
➜  ~ tcpdump -i flannel.1 -e -nn
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on flannel.1, link-type EN10MB (Ethernet), capture size 262144 bytes
11:26:46.822551 f6:f2:b0:fa:75:73 > da:a9:48:cb:15:3b, ethertype IPv4 (0x0800), length 74: 10.244.0.0.5326 > 10.244.1.8.9000: Flags [S], seq 2320302970, win 29200, options [mss 1460,sackOK,TS val 2082762796 ecr 0,nop,wscale 7], length 0
```

在我们发送请求时看到：121.4.190.84.59814 > 10.0.20.11.8472

121.4.190.84是我们的主节点公网IP

10.0.20.11是我们的工作节点的内网ip，这个应该是腾讯云配置的从公网ip转内网IP



eth0的10.244.0.0.14571 > 10.244.1.8.9000和flannel的10.244.0.0.5326 > 10.244.1.8.9000，应该是flannel解包后发起请求

10.224.0.0是路由网卡，如下：

```sh
➜  ~ ip route
10.244.0.0/24 via 10.244.0.0 dev flannel.1 onlink
```

具体的解包，这个东西目前博主还不太清楚，等后面搞清楚了再来说说

目前是经过flannel解包后，得到了目标pod，然后经过路由，请求到了pod中



到此请求链路就完成了



###### 7，8，9响应返回

在flannel的包中：

```sh
➜  ~ tcpdump -i flannel.1 -e -nn
11:26:46.822637 da:a9:48:cb:15:3b > f6:f2:b0:fa:75:73, ethertype IPv4 (0x0800), length 74: 10.244.1.8.9000 > 10.244.0.0.5326: Flags [S.], seq 3013507867, ack 2320302971, win 27960, options [mss 1410,sackOK,TS val 1030036288 ecr 2082762796,nop,wscale 7], length 0
```

可以看到我们的应用处理完成后，将请求回到了flannel：10.244.1.8.9000 > 10.244.0.0.5326



工作节点的物理网卡中,在我们发起请求的时候，向我们的主节点的8472端口返回了响应

```sh
➜  ~ tcpdump -i eth0 -e -nn |grep 121.4.190.84.8472
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
11:43:33.880431 52:54:00:4f:72:ca > fe:ee:b8:5a:5a:63, ethertype IPv4 (0x0800), length 124: 10.0.20.11.40470 > 121.4.190.84.8472: OTV, flags [I] (0x08), overlay 0, instance 1
```



###### 10，11主节点响应接收

下面是主节点物理网卡的

```sh
➜  tcpdump -i eth0 -e -nn port 8472
11:45:44.170343 fe:ee:b5:81:6a:26 > 52:54:00:f4:44:15, ethertype IPv4 (0x0800), length 116: 106.55.227.160.48943 > 172.17.16.14.8472: OTV, flags [I] (0x08), overlay 0, instance 1
da:a9:48:cb:15:3b > f6:f2:b0:fa:75:73, ethertype IPv4 (0x0800), length 66: 10.244.1.8.9000 > 10.244.0.0.53794: Flags [.], ack 107, win 219, options [nop,nop,TS val 1031173621 ecr 2083900128], length 0
```

106.55.227.160.48943 > 172.17.16.14.8472：工作节点的公网ip，请求到主节点的内网ip的8472端口



下面是主节点的flannel的

```sh
➜ tcpdump -i flannel.1 -e -nn
11:49:13.675481 da:a9:48:cb:15:3b > f6:f2:b0:fa:75:73, ethertype IPv4 (0x0800), length 74: 10.244.1.8.9000 > 10.244.0.0.54626: Flags [S.], seq 3933634312, ack 2648191468, win 27960, options [mss 1410,sackOK,TS val 1031383126 ecr 2084109635,nop,wscale 7], length 0
```



响应的返回其实就是请求的逆向，理解了请求的链路，响应的链路大同小异：

请求：发起者的本地ip+随机端口 --> 目标IP+目标端口

响应：目标IP+目标端口 --> 发起者的本地ip+随机端口



## 总结

写本篇文章，耗费了很多精力，但也收获很大，感觉经过这次的问题解决，自己对网络部分又加深了理解

查询资料中，资料大部分是局域网进行集群部署的，但使用公网ip部署，有一些差异，有一些跨主机解决方案是用不了的，比如：

- Flannel的Host Gateway
- Calico的BGP

目前尝试下来，vxlan还是可以，但其关键的一步就是公网ip网卡的设置

还有就是相关端口的开放，比如这里的8472端口



知道整个链路，后序排查问题就有思路，博主也希望这篇详细的链路分析能给各位一些问题排查上的帮助

这部分写了也四五天了，推荐两本好书，如果各位遇到了问题，可以去里面多看看，看看能不能给点启发

- 《Kubernetes网络权威指南：基础、原理与实践》：原理和实践都有，贼好，这周遇到问题就翻......
- 《网络是怎样连接的》：基础原理的，如果局域网内的机器如何使用使用同一个公网ip进行网络访问，如果不了解上面的工作节点的公网ip，请求到主节点的内网ip的8472端口，得看看这个

两本书在微信读书上面都有，不得不说，现在这个时代获取知识真方便，真好



后面有一些遇到问题的记录，如果遇到了相似的问题，也可以看看



## 问题记录与解决方法
### 重新安装CNI插件后，node节点一直notready
查看kubelet日志，一直报错如下：

```sh
m runtime service failed" err="rpc error: code = Unknown desc = failed to destroy network for sandbox \"f8b273d0b213af9a143f5729da56365efaf8fb322a1299ce2cfb8eef2cbd03c2\": cni plugin not initialized" podSandboxID="f8b273d0b213af9a14
dbox before removing" err="rpc error: code = Unknown desc = failed to destroy network for sandbox \"f8b273d0b213af9a143f5729da56365efaf8fb322a1299ce2cfb8eef2cbd03c2\": cni plugin not initialized" sandboxID="f8b273d0b213af9a143f5729d
m runtime service failed" err="rpc error: code = Unknown desc = failed to destroy network for sandbox \"e6a73ececa384f6946f9d764a093f639901fc647d84cd32f0175505a72b50f47\": cni plugin not initialized" podSandboxID="e6a73ececa384f6946
dbox before removing" err="rpc error: code = Unknown desc = failed to destroy network for sandbox \"e6a73ececa384f6946f9d764a093f639901fc647d84cd32f0175505a72b50f47\": cni plugin not initialized" sandboxID="e6a73ececa384f6946f9d764a
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
k not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
```

解决办法，重新启动下contained：

```sh
systemctl daemon-reload
systemctl restart containerd
```



### Readiness probe failed: calico/node is not ready: BIRD is not ready: Error querying BIRD: unable to connect to BIRDv4 socket: dial unix /var/run/calico/bird.ctl: connect: connection refused

确实一些组件导致的，下载后，重启kubelet即可



```sh
# 如果服务器内下载不了，访问这个页面去下最新的：https://github.com/containernetworking/plugins/releases，后面手动上传后解压
wget https://github.com/containernetworking/plugins/releases/download/v1.0.1/cni-plugins-linux-arm64-v1.0.1.tgz
tar axvf ./cni-plugins-linux-arm64-v1.0.1.tgz  -C /opt/cni/bin/

systemctl restart kublet
```



## 参考链接

- [使用 Service 暴露你的应用](https://kubernetes.io/zh-cn/docs/tutorials/kubernetes-basics/expose/expose-intro/)
- [Kubernetes系列之五：使用yaml文件创建service向外暴露服务](https://blog.csdn.net/wucong60/article/details/81699196)
- [Kubernetes对象之Service](https://www.cnblogs.com/tylerzhou/p/10989881.html)
- [linux ip 转发设置 ip_forward、ip_forward与路由转发](https://blog.csdn.net/li_101357/article/details/78416813)
- [calico Overlay networking](https://projectcalico.docs.tigera.io/networking/vxlan-ipip#configure-vxlan-encapsulation-for-all-inter-workload-traffic)
- [k8s中正确删除一个pod](https://www.cnblogs.com/effortsing/p/10496547.html)
- [强制删除POD](https://www.jianshu.com/p/fe7473e43d76)
- [tcpdump详解和定位解决问题实例分析](https://blog.csdn.net/wj31932/article/details/106570542)
- [iptables一次关于主机发出数据包的DNAT操作经验](https://blog.csdn.net/m0_37549390/article/details/110865635)
- [关于iptables添加规则不生效的问题](https://blog.csdn.net/donglynn/article/details/73530542)
- [关于grep命令的or，and，not操作的例子](https://blog.csdn.net/jackaduma/article/details/6900242)
- [linux -- 添加、修改、删除路由](https://www.cnblogs.com/hf8051/p/4530906.html)
- [iptables一次关于主机发出数据包的DNAT操作经验](https://blog.csdn.net/m0_37549390/article/details/110865635)
- [iptables: change local source address if destination address matches](https://unix.stackexchange.com/questions/243451/iptables-change-local-source-address-if-destination-address-matches)
- [What is the difference between NAT OUTPUT chain and NAT POSTROUTING chain?](https://unix.stackexchange.com/questions/402233/what-is-the-difference-between-nat-output-chain-and-nat-postrouting-chain)
- [[kubernetes] 跨云厂商使用公网IP搭建k8s v1.20.9集群](https://blog.51cto.com/xiaowangzai/5167661)
- [解决CNI报failed to find plugin "bridge" in path [/opt/cni/bin]错误](https://vqiu.cn/k8s-cni-failed-to-find-plugein-bridge/)
- [CNI plugins download](https://github.com/containernetworking/plugins/releases)
- [tcpdump详解和定位解决问题实例分析](https://blog.csdn.net/wj31932/article/details/106570542)



## 最后

我正在参与掘金技术社区创作者签约计划招募活动，[点击链接报名投稿](https://juejin.cn/post/7112770927082864653)。