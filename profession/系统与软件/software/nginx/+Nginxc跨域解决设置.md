# Nginx跨域解决配置示例
***

这是我参与2022首次更文挑战的第25天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7052884569032392740)

## 简介
在日常学习和工作开发中，需要请求两个不同配置的请求经常存在，本文介绍如果还使用Nginx配置解决其跨域问题

## 相关理论
首先需要了解什么是跨域，下面的两个文章说的很好，请仔细阅读后，然后自己去动手尝试配置，将有深刻的体会

- [不要再问我跨域的问题了](https://segmentfault.com/a/1190000015597029)
- [【译】3种解决CORS错误的方式与Access-Control-Allow-Origin的作用原理](https://segmentfault.com/a/1190000022506474)
- [Nginx实战（九）跨域配置（解决CORS报错）](https://blog.csdn.net/ouyida3/article/details/86771683)
- [nginx代理跨域配置add_header Access-Control-Allow-Origin 不生效的解决方法](https://yijiebuyi.com/blog/ba792f47a3e713e3c63082587a6d8675.html)

总结来说：

- 跨域是浏览器的安全策略造成的，但其也是必要的，不能为了方便而放弃安全性
- 跨域是不同源的请求导致的：IP、域名、端口等不同都会造成跨域
- 跨域的判断是由请求头和响应头的相关字段进行判断的，这个是设置的基础

那解决方法目前看来有三个：

- 前端层面自己解决：前端请求时自己进行代理
- 网关层面进行解决：在nginx、kong同统一网关中进行配置解决
- 服务后台解决：在Go、Java Web中进行配置解决，经典的Cors配置

但注意的是，有时候是只能使用一样跨域解决方式的，最终的效果是需要保证最后前端收到的请求头符合规范,特别是下面这个头：

Access-Control-Allow-Origin: *

如果返回的响应头里面少了会跨域，但多了，比如返回了两个相同的，也会跨域

注：在实际开发中，如果发生跨域，排查的第一步就是看看响应头里面是否返回了正确的数据，这样就能精确的进行下一步的解决操作

## 配置示例
### 场景描述
场景如下：

前端需要请求两个服务：

- http://www.service1.com/getxxx
- http://www.service2.com/getxxx

而页面的访问链接是：http://www.web.com

如果我们不进行任何配置，通过浏览器访问页面，两个服务的请求都会报：Cors Err

注：这里就没有示例代码了，自己可以用vue简单写写，然后访问

### Nginx配置
下面展示如果通过配置Nginx解决跨域问题：

配置两个服务，监听在相同的端口，服务名不同而已，在转发配置中加上跨域配置，示例文件如下：


```nginx
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    # 转发websocket需要的设置
    proxy_set_header X-Real_IP $remote_addr;
    proxy_set_header Host $host;
    proxy_set_header X_Forward_For $proxy_add_x_forwarded_for;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';

    # Service1服务配置
    server {
        listen       80;
        server_name  www.service1.com;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
            proxy_pass http://localhost:8188/;
            # 设置是否允许 cookie 传输
            add_header Access-Control-Allow-Credentials true;
            # 允许请求地址跨域 * 做为通配符
            add_header Access-Control-Allow-Origin * always;
            # 允许跨域的请求方法
            add_header Access-Control-Allow-Methods 'GET, POST, PUT, DELETE, OPTIONS';
            add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';

            if ($request_method = 'OPTIONS') {
                return 204;
            }
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }

    # Service2的配置
    server {
        listen       80;
        server_name  www.service2.com;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
            proxy_pass http://localhost:9096/;
            add_header Access-Control-Allow-Origin * always;
            add_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS,PUT,DELETE' always;
            add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';

            if ($request_method = 'OPTIONS') {
                return 204;
            }
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

### 浏览器访问确认
F12打开调试工具，在请求中，会开到响应头和请求头都会有下面的字段：

请求头：

```
GET /route/xxxxxx
Host: host
Connection: keep-alive
Access-Control-Allow-Origin: *
Accept: application/json, text/plain, */*
Authorization: xxxxxxxxxx
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/90.0.4430.85 Safari/537.36
Origin: http://xxx.com
Referer: http://xxx.com
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
```

相应头：

```
HTTP/1.1 200 OK
Server: nginx/1.14.1
Date: Sat, 24 Apr 2021 15:17:10 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET,POST,OPTIONS,PUT,DELETE
Access-Control-Allow-Headers: DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization
```

## 总结
本篇介绍了如果通过nginx网关配置去解决跨域问题，需要在请求中添加相关的头即可

最后再说明排查的第一步，看看出现跨域问题时，返回的响应头是否符合规范，重点是下面两个，其他的也需要关注，但目前遇到的问题，基本都是下面两个返回不规范导致的：

- Access-Control-Allow-Origin: *
- Access-Control-Allow-Methods: GET,POST,OPTIONS,PUT,DELETE