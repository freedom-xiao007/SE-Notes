# Redis Cluster 配置
***
## 简介
&ensp;&ensp;&ensp;&ensp;在 Centos8 上部署 redis Cluster

*这里就写一个大概，比较好的，可以参考后面的链接，进行更详细，完整的设置*

### 配置详情与记录

```shell
# 安装
dnf module install redis -y

# 修改配置文件
cp /etc/redis.conf /etc/redis.conf.orig
vi /etc/redis.conf

# 修改内容如下
bind  10.42.0.247
protected-mode no
port 6379
cluster-enabled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 15000

# 启动
systemctl restart redis
cat /var/log/redis/redis.log

# 下面的6380,6381,6382,6383,6384按照上面配置好以后，进行下面命令
redis-cli --cluster create 192.168.101.105:6379 192.168.101.105:6380 192.168.101.105:6381 192.168.101.105:6382 192.168.101.105:6383 192.168.101.105:6384 --cluster-replicas
redis-cli -h 192.168.101.105 -p 6379 cluster nodes
redis-cli -h 192.168.101.105 -p 6379 cluster nodes  | grep master
redis-cli -h 192.168.101.105 -p 6379 cluster nodes  | grep slave
```

## 参考链接
- [Windows下部署redis主从、哨兵（sentinel）、集群（cluster）](https://blog.csdn.net/baidu_27627251/article/details/112143714)
- [配置 redis 的主从复制，sentinel 高可用，Cluster 集群](https://github.com/Johar77/JAVA-000/tree/main/Week_12)