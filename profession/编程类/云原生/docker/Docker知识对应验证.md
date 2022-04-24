# Docker知识对应验证

***

## 简介

## 网络部分
```powershell
docker run -tid --name centos8 centos:8 bash

$ docker exec -ti centos8 traceroute 114.114.114.114
traceroute to 114.114.114.114 (114.114.114.114), 30 hops max, 60 byte packets
 1  172.17.0.1 (172.17.0.1)  0.046 ms  0.007 ms  0.005 ms
 2  192.168.1.1 (192.168.1.1)  4.210 ms  4.245 ms  4.235 ms
 3  100.64.0.1 (100.64.0.1)  7.034 ms  7.052 ms  7.044 ms
 4  118.116.49.185 (118.116.49.185)  12.602 ms  12.624 ms *
 5  171.208.196.5 (171.208.196.5)  7.995 ms 171.208.199.249 (171.208.199.249)  9.718 ms 61.139.121.49 (61.139.121.49)  8.169 ms
 6  202.97.76.138 (202.97.76.138)  36.394 ms * *
 7  * * *
 8  * * *
 9  * * *
10  * * *
11  * * *
12  * * *
13  * * *
14  * * *
15  * * *
16  * * *
17  * * *
```

```powershell
$ ip addr
4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:d4:fb:76:d9 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:d4ff:fefb:76d9/64 scope link
       valid_lft forever preferred_lft forever
6: veth0cf8b9a@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether 42:da:8b:c0:20:7f brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::40da:8bff:fec0:207f/64 scope link
       valid_lft forever preferred_lft forever

$ docker exec -ti centos8 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
5: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

```powershell
$ iptables  -t  nat  -nL --line-numbers
Chain PREROUTING (policy ACCEPT)
num  target     prot opt source               destination
1    DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination
1    DOCKER     all  --  0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT)
num  target     prot opt source               destination
1    MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0

Chain DOCKER (2 references)
num  target     prot opt source               destination
1    RETURN     all  --  0.0.0.0/0            0.0.0.0/0
```
