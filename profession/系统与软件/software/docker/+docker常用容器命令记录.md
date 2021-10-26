# 开发中Docker常用容器记录
***
## 概览
分享工作学习中常用的Docker容器使用：

- 比如常用数据库的使用
- 消息队列类的使用
- 用于服务发现的容器使用
- 还有其他工作学习中使用到的

持续更新：https://juejin.cn/post/6965321555475693582/

## 数据库

```sh
docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root -d mysql:latest
docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root -d mysql:latest --lower_case_table_names=1


docker run -dit --name neo4j -p 7474:7474 -p 7687:7687 --env=NEO4J_AUTH=none neo4j:3.5.13

docker run -dit --name redis -p 6379:6379 redis
docker run -d --name redis -p 6379:6379 redis --requirepass "password"
./redis-cli -h 127.0.0.1 -p 6379 -a myPassword

docker run -dit --name mongo -p 27017:27017 mongo
# windows
docker run  -d `
  -e MONGO_INITDB_ROOT_USERNAME=mongoadmin`
  -e MONGO_INITDB_ROOT_PASSWORD=secret `
  -v /ect/localtime:/etc/localtime`
  -p 3389:27017 `
  --name mongodb`
  --restart always mongo:latest

docker run  -d -e MONGO_INITDB_ROOT_USERNAME=mongoadmin -e MONGO_INITDB_ROOT_PASSWORD=secret -v /etc/localtime:/etc/localtime:ro -p 3389:27017  --name mongodb --restart always mongo:latest
# linux
docker run  -d \
  -e MONGO_INITDB_ROOT_USERNAME=mongoadmin \
  -e MONGO_INITDB_ROOT_PASSWORD=secret \
  -v /root/docker/mongodb/data:/data/  \
  -p 27017:27017 \
  --network mongo-network \
  --name mongo \
  --restart always \
  mongo:latest
```

### ES
```yaml
version: "3.7"
services:
    kibana:
        image: kibana:7.4.2
        container_name: kibana
        environment:
            - ELASTICSEARCH_HOSTS=http://es01:9200
        ports:
            - 5601:5601
        networks:
            - elastic
    es01:
        image: elasticsearch:7.4.2
        environment:
            - node.name=es01
            - cluster.name=es-docker-cluster
            - network.host=_local_,_site_
            - network.publish_host=_local_
            - discovery.type=single-node
            - bootstrap.memory_lock=true
            - "ES_JAVA_OPTS=-Xms750m -Xmx750m"
            - "ELASTIC_PASSWORD=password"
        ulimits:
            memlock:
                soft: -1
                hard: -1
            nofile:
                soft: 65535
                hard: 65535
        ports:
            - 9200:9200
            - 9300:9300
        networks:
            - elastic
networks:
  elastic:
    driver: bridge
```

使用过程中需要安装IK分词

```shell script
// 进入容器es
docker exec -it es /bin/bash
// 使用bin目录下的elasticsearch-plugin install安装ik插件,注意与ES版本的一致性
bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.4.2/elasticsearch-analysis-ik-7.4.2.zip
// 再重启下容器
docker restart es
```

- [2.4-docker es安装插件](https://www.jianshu.com/p/275e629535f6)
- [ES - 基于Golang的简单使用ElasticSearch](tkstorm.com/posts-list/software-engineering/elastic/es-docker/)
- [Elasticsearch —— docker部署+ik分词器](https://www.jianshu.com/p/d8b0c736070f)

## 服务发现
### Nacos
```shell
git clone https://github.com/nacos-group/nacos-docker.git
cd nacos-docker
docker-compose -f example/standalone-derby.yaml up

http://localhost:8848/nacos/#/login

用户名和密码
nacos
nacos
```

### etcd
```shell script
NODE1=192.168.1.21
DATA_DIR="etcd-data"
# v3
REGISTRY=gcr.io/etcd-development/etcd

docker run \
  -p 2379:2379 \
  -p 2380:2380 \
  --volume=${DATA_DIR}:/etcd-data \
  --name etcd ${REGISTRY}:latest \
  /usr/local/bin/etcd \
  --data-dir=/etcd-data --name node1 \
  --initial-advertise-peer-urls http://${NODE1}:2380 --listen-peer-urls http://0.0.0.0:2380 \
  --advertise-client-urls http://${NODE1}:2379 --listen-client-urls http://0.0.0.0:2379 \
  --initial-cluster node1=http://${NODE1}:2380



docker run -p 2379:2379 -p 2380:2380 --name etcd quay.io/coreos/etcd:latest /usr/local/bin/etcd --data-dir=/etcd-data --name node1 --initial-advertise-peer-urls http://192.168.101.112:2380 --listen-peer-urls http://0.0.0.0:2380 --advertise-client-urls http://192.168.101.112:2379 --listen-client-urls http://0.0.0.0:2379 --initial-cluster node1=http://192.168.101.112:2380

docker run -dit --name etcd_web -p 8081:8080 evildecay/etcdkeeper
```

### zookeeper
```shell script
docker run -dit --name zk -p 2181:2181 zookeeper
```

### Consul
```shell script
# linux
docker run \
    -d \
    -p 8500:8500 \
    -p 8600:8600/udp \
    --name=consul_badger \
    consul agent -server -ui -node=server-1 -bootstrap-expect=1 -client=0.0.0.0

# windows
docker run `
    -d `
    -p 8500:8500 `
    -p 8600:8600/udp `
    --name=consul_badger `
    -v D:\Docker\compose\consul:/consul/config `
    consul agent -server -ui `-node`=server-1 `-bootstrap-expect`=1 `-client`=0.0.0.0


# ACL config in /consul/config
{
	"datacenter": "dc1",
	"bootstrap_expect": 1,
	"log_level": "INFO",
	"node_name": "consul_server_1",
	"client_addr": "0.0.0.0",
	"server": true,
	"ui": true,
	"enable_script_checks": true,
	"addresses": {
	    "https": "0.0.0.0",
	    "dns": "0.0.0.0"
	}
}

primary_datacenter = "dc1"
acl {
  enabled = true
  default_policy = "deny"
  enable_token_persistence = true
  tokens { 
    master = "474dcea7-ee4d-3f11-1af1-a38eb37d3f5d"
  }
}
```

## 消息队列
### docker-activemq
```shell script
docker run -dit --name activemq -p 11616:61616 -p 8161:8161 -p 1883:1883 rmohr/activemq
```

初始账号：
admin admin
user user

### docker rabbitmq
```shell script
docker run -dit --name rabbitmq -p 5672:5672 rabbitmq

docker run -d -p 5672:5672 -p 15672:15672 --name rabbitmq rabbitmq:management

http://127.0.0.1:15672

用户和密码：
guest
guest
```

## 其他
### Nginx
```shell script
docker run -dit --name nginx --restart=always -p 80:80 -v D:/Docker/images/nginx/:/etc/nginx/ -v D:/Docker/images/nginx/download:/home/download/ nginx

docker run -dit --name nginx -p 80:80 -v D:/Docker/images/nginx/config/:/etc/nginx/ -v D:/Docker/images/nginx/download:/home/download/ nginx
```

### 前后端接口管理：Yapi
```shell script
git clone https://gitee.com/fjc0k/docker-YApi.git
docker-compose up -d

然后，通过 http://localhost:40001 即可访问 YApi。
用户密码在docker-compose.yml中，YAPI_ADMIN_ACCOUNT 为你的管理员邮箱，YAPI_ADMIN_PASSWORD 为你的管理员密码
```

### Kong
- [Documentation for Kong Gateway (OSS)](https://docs.konghq.com/gateway-oss/2.2.x/)
- [Docker Installation](https://docs.konghq.com/install/docker/?_ga=2.15676593.643431049.1616122322-97521002.1616122322)

```shell script
# linux
docker run -d --name kong \
     --network=kong-net \
     -v "kong-vol:/usr/local/kong/declarative" \
     -e "KONG_DATABASE=off" \
     -e "KONG_DECLARATIVE_CONFIG=/usr/local/kong/declarative/kong.yml" \
     -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
     -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
     -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
     -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
     -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
     -p 8000:8000 \
     -p 8443:8443 \
     -p 127.0.0.1:8001:8001 \
     -p 127.0.0.1:8444:8444 \
     kong:latest


# windows
docker network create kong-net

docker run -d --name kong-database `
               --network=kong-net `
               -p 5432:5432 `
               -e "POSTGRES_USER=kong" `
               -e "POSTGRES_DB=kong" `
               -e "POSTGRES_PASSWORD=kong" `
               postgres:9.6

docker run --rm `
     --network=kong-net `
     -e "KONG_DATABASE=postgres" `
     -e "KONG_PG_HOST=kong-database" `
     -e "KONG_PG_PASSWORD=kong" `
     -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" `
     kong:latest kong migrations bootstrap

docker run -d --name kong `
     --network=kong-net `
     -e "KONG_DATABASE=postgres" `
     -e "KONG_PG_HOST=kong-database" `
     -e "KONG_PG_PASSWORD=kong" `
     -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" `
     -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" `
     -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" `
     -e "KONG_PROXY_ERROR_LOG=/dev/stderr" `
     -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" `
     -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" `
     -p 8000:8000 `
     -p 8443:8443 `
     -p 8001:8001 `
     -p 8444:8444 `
     kong:latest

curl -i http://localhost:8001/

docker run --rm pantsel/konga:latest -c prepare -a postgres -u postgresql://kong:kong@192.168.110.242:5432/konga
     
docker run -p 1337:1337 `
        --network kong-net `
        --name konga `
        -e "NODE_ENV=production"  `
        -e "DB_ADAPTER=postgres" `
        -e "DB_URI=postgresql://kong:kong@192.168.110.242:5432/konga" `
        pantsel/konga
```

### Portainer - Docker的可视化管理工具使用详解
```
# linux
docker run -d -p 9000:9000 --name portainer --restart always -v D:/temp/docker/portainer/docker.sock:/var/run/docker.sock -v D:/temp/docker/portainer/data:/data portainer/portainer

# windows
docker run -d -p 9000:9000 --name portainer --restart always -v -v D:/temp/docker/portainer/docker_engine:/var/run/pipe/docker_engine -v D:/temp/docker/portainer/data:/data portainer/portainer

用户和密码：
admin password
```

### Centos
```
docker run -tid --name centos8 -p 82:80 -v D:/temp/centos/:/root/ centos
```

### 本地镜像仓库
```
docker run -d -p 5000:5000 --name registry --restart=always -v D:/Docker/registry:/var/lib/registry registry:latest

# 在docker deamon 中加入 127.0.0.1:5000

# 上传
docker tag xxxx:test 192.168.110.196:5000/xxxx:test
docker push 192.168.110.196:5000/xxxx:test

# 下载
docker pull 192.168.110.196:5000/xxxx:test
```

- [docker 搭建本地私有仓库及镜像上传时HTTPS client问题解决(windows 10)](https://blog.csdn.net/qq_2300688967/article/details/83545647)
- [私有仓库](https://yeasy.gitbook.io/docker_practice/repository/registry)