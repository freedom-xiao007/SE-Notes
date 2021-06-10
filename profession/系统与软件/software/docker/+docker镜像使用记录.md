# Docker 镜像使用记录
***
## MySQL
### 使用记录
- 拉取官方镜像：docker pull mysql
- 运行镜像：docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root -d mysql:latest
    + -p:将容器的3306端口映射到宿主机的3306端口
    + -e:设置mysql的root用户的密码为root

- 遇到sock之类的错误，删文件，在重启后再重启

## Neo4j
```shell script
docker run -dit --name neo4j -p 7474:7474 -p 7687:7687 --env=NEO4J_AUTH=none neo4j:3.5.13
```

## Redis
```shell script
docker run -dit --name redis -p 6379:6379 redis

docker run -d --name redis -p 6379:6379 redis --requirepass "password"
```

## MongoDB
```shell script
docker run -dit --name mongo -p 27017:27017 mongo
```

## Nginx
```shell script
docker run -dit --name nginx -p 80:80 -v D:/temp/nginx.conf:/etc/nginx/nginx.conf nginx
```

## docker-activemq
```shell script
docker run -dit --name activemq -p 11616:61616 -p 8161:8161 -p 1883:1883 rmohr/activemq
```

初始账号：
admin admin
user user

## docker rabbitmq
```shell script
docker run -dit --name rabbitmq -p 5672:5672 rabbitmq
```

### zookeeper
```shell script
docker run -dit --name zk -p 2181:2181 zookeeper
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
docker run -d --name kong `
     --network=kong-net `
     -v "kong-vol:/usr/local/kong/declarative" `
     -e "KONG_DATABASE=off" `
     -e "KONG_DECLARATIVE_CONFIG=/usr/local/kong/declarative/kong.yml" `
     -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" `
     -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" `
     -e "KONG_PROXY_ERROR_LOG=/dev/stderr" `
     -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" `
     -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" `
     -p 8000:8000 `
     -p 8443:8443 `
     -p 127.0.0.1:8001:8001 `
     -p 127.0.0.1:8444:8444 `
     kong:latest
```

## 参考链接
- [mysql](https://hub.docker.com/_/mysql)