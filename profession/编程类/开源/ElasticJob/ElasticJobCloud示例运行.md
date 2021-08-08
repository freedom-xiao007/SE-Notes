# ElasticJobCloud示例运行
***
## 简介
工作中需要用到分布式任务调度相关的东西，于是探索下目前开源的分布式任务调度组件，本篇就对ElasticJobCloud尝尝鲜

## 部署与运行
Shardingsphere的东西风格还真是一致，和分布式的数据库一样，资料简洁，没有相关的知识，看的是一脸懵，难搞，但还是得硬着头皮搞

话不多说，开搞。提示：由于是本地示例，大部分都是单机部署，集群请自行查阅资料补充

### 1.下载源码
到GitHub上下载最新的源码到本地： [https://github.com/apache/shardingsphere-elasticjob](https://github.com/apache/shardingsphere-elasticjob)

我们开始尝试运行模块： examples -- elasticjob-example-cloud

在示例运行说明文件中说明了要运行的其他相关组件：zookeeper、Mesos等待，先去部署下这些组件

### 2.部署Zookeeper
Zookeeper简单运行下面的命令进行部署即可：

```shell script
docker run -dit --name zk -p 2181:2181 zookeeper
```

### 3.部署Mesos


```shell script
# linux下换行使用 \ ， windows 下换行使用 `
docker run -d --net=host \
  --name mesos-master \
  -e "MESOS_HOSTNAME=mesos.online" \
  -e MESOS_PORT=5050 \
  -e MESOS_ZK=zk://127.0.0.1:2181/mesos \
  -e MESOS_QUORUM=1 \
  -e MESOS_REGISTRY=in_memory \
  -e MESOS_LOG_DIR=/var/log/mesos \
  -e MESOS_WORK_DIR=/var/tmp/mesos \
  -v "/var/log/mesos:/var/log/mesos" \
  -v "/var/tmp/mesos:/var/tmp/mesos" \
  mesosphere/mesos-master:1.5.2

docker run -d --net=host `
  --name mesos-master `
  -e "MESOS_HOSTNAME=mesos.online" `
  -e MESOS_PORT=5050 `
  -e MESOS_ZK=zk://127.0.0.1:2181/mesos `
  -e MESOS_QUORUM=1 `
  -e MESOS_REGISTRY=in_memory `
  -e MESOS_LOG_DIR=/var/log/mesos `
  -e MESOS_WORK_DIR=/var/tmp/mesos `
  -v "/var/log/mesos:/var/log/mesos" `
  -v "/var/tmp/mesos:/var/tmp/mesos" `
  mesosphere/mesos-master:1.5.2
```

```shell script
# linux
docker run -d --net=host \
  --name mesos-slave \
  --privileged \
  -e MESOS_PORT=5051 \
  -e MESOS_MASTER=zk://127.0.0.1:2181/mesos \
  -e MESOS_SWITCH_USER=0 \
  -e MESOS_CONTAINERIZERS=docker,mesos \
  -e MESOS_LOG_DIR=/var/log/mesos \
  -e MESOS_WORK_DIR=/var/tmp/mesos \
  -e MESOS_SYSTEMD_ENABLE_SUPPORT=false \
  -v "/var/log/mesos-sl:/var/log/mesos" \
  -v "/var/tmp/mesos-sl:/var/tmp/mesos" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /cgroup:/cgroup \
  -v /sys:/sys \
  -v /usr/local/bin/docker:/usr/local/bin/docker \
  mesosphere/mesos-slave:1.5.2

# windows
docker run -d --net=host `
  --name mesos-slave `
  --privileged `
  -e MESOS_PORT=5051 `
  -e MESOS_MASTER=zk://127.0.0.1:2181/mesos `
  -e MESOS_SWITCH_USER=0 `
  -e MESOS_CONTAINERIZERS=docker,mesos `
  -e MESOS_LOG_DIR=/var/log/mesos `
  -e MESOS_WORK_DIR=/var/tmp/mesos `
  -e MESOS_SYSTEMD_ENABLE_SUPPORT=false `
  -v "/var/log/mesos-sl:/var/log/mesos" `
  -v "/var/tmp/mesos-sl:/var/tmp/mesos" `
  -v /var/run/docker.sock:/var/run/docker.sock `
  -v /cgroup:/cgroup `
  -v /sys:/sys `
  -v /usr/local/bin/docker:/usr/local/bin/docker `
  mesosphere/mesos-slave:1.5.2
```

```shell script
# linux
docker run --net=host -d \
  --name marathon \
  mesosphere/marathon \
  --master zk://127.0.0.1:2181/mesos \
  --zk zk://127.0.0.1:2181/marathon

# windows
docker run --net=host -d `
  --name marathon `
  mesosphere/marathon `
  --master zk://127.0.0.1:2181/mesos `
  --zk zk://127.0.0.1:2181/marathon
```