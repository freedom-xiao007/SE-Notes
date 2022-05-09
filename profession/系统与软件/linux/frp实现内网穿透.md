# FRP 实现内网穿透
***

安装ssh服务端，如果是root登录，确认开启了root登录
```sh
sudo apt-get install openssh-server
```

```sh
wget https://github.91chi.fun//https://github.com//fatedier/frp/releases/download/v0.39.1/frp_0.39.1_linux_amd64.tar.gz

mkdir /opt/frp
tar -C /opt/frp -xzf frp_0.39.1_linux_amd64.tar.gz

cd /opt/frp/frp_0.39.1_linux_amd64/

vi frps.ini

[common]
bind_port = 7000
dashboard_port = 7500
dashboard_user = xxx
dashboard_pwd = xxx
vhost_http_port = 10080

说明：
bind_port：frp客户端和服务端连接的端口；
dashbaord_port,user,pwd：分别为控制面板的web端口和登录的用户和密码；
vhost_http_port：反向代理http主机端口，则即通过vps_ip:vhost_http_port访问内网web服务
若使用https：vhost_https_port = 10443

[root@hecs-280581 frp_0.39.1_linux_amd64]# ./frps -c ./frps.ini
2022/03/08 21:05:02 [I] [root.go:200] frps uses config file: ./frps.ini
2022/03/08 21:05:02 [I] [service.go:193] frps tcp listen on 0.0.0.0:7000
2022/03/08 21:05:02 [I] [service.go:292] Dashboard listen on 0.0.0.0:7500
2022/03/08 21:05:02 [I] [root.go:209] frps started successfully

http://ip_address:7500/static/#/

nohup ./frps -c ./frps.ini &    #后台运行
```

```sh
wget https://github.91chi.fun//https://github.com//fatedier/frp/releases/download/v0.39.1/frp_0.39.1_linux_amd64.tar.gz

mkdir /opt/frp
tar -C /opt/frp -xzf frp_0.39.1_linux_amd64.tar.gz

cd /opt/frp/frp_0.39.1_linux_amd64/
 
 vi frpc.ini

 ➜  frp_0.39.1_linux_amd64 cat frpc.ini
[common]
server_addr = xxxxx
server_port = 7000

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000


➜  frp_0.39.1_linux_amd64 ./frpc -c ./frpc.ini
2022/03/08 21:12:00 [I] [service.go:327] [0ee2c60c51163b75] login to server success, get run id [0ee2c60c51163b75], server udp port [0]
2022/03/08 21:12:00 [I] [proxy_manager.go:144] [0ee2c60c51163b75] proxy added: [ssh]
2022/03/08 21:12:00 [I] [control.go:181] [0ee2c60c51163b75] [ssh] start proxy success


nohup ./frpc -c ./frpc.ini &    #后台运行
```

腾讯云服务器开放端口有些问题，有时候开通无效

华为云开通正常

## 参考链接
- [vps+frp实现内网穿透](https://wxt123.top/2021/01/28/vps+frp%E5%AE%9E%E7%8E%B0%E5%86%85%E7%BD%91%E7%A9%BF%E9%80%8F/)
- [frp](https://github.com/fatedier/frp)
- [ubuntu开启SSH服务远程登录](https://blog.csdn.net/jackghq/article/details/54974141)