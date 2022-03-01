# JetBrains Gateway 初体验
***

## 简介
本文将初步探索JetBrains Gateway的使用，如果在本地使用Gateway，使用远程Linux服务器的Goland来编写代码

## 设置Linux服务器
老手可大致浏览即可，可快速跳过

首先登录Linux服务器，进行下面的操作

### 登录用户设置
首先得开启相关的SSH登录权限，本文中使用密码进行登录

可以新建一个用户，不用默认的root用户

```sh
sudo useradd lw
sudo passwd lw
```

如果是使用root用户，需要注意使用root登录是否开启

```sh
vim /etc/ssh/sshd_config
# 如果 #PermitRootLogin yes 被注释掉了，需要去掉#号，打开

# 保存更改后，重启ssh
service sshd restart

# 提前建立工程目录
mkdir -p code/go/dockerDemo
```

### 安装Golang环境
大致命令如下：

本文进行全局的安装配置

```sh
mkdir download
mkdir opt
cd download
wget https://go.dev/dl/go1.17.7.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.17.7.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
go version
```

这样，准备工作差不多都完成了

## JetBrains Gateway 连接
### 连接登录
打开Gateway，输入自己对应的用户名和服务器ip，输入相应的密码

![屏幕截图 2022-03-02 055054.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/09ac5603e37b4d12baac1074080700e6~tplv-k3u1fbpfcp-watermark.image?)

### 打开工程
选择对应的IDE，这里选择Goland，然后选择上面我们建立好的工程目录

安装和准备工作需要较长的时间，需要耐心等候

![屏幕截图 2022-03-02 055725.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d182c8ad2392456d883d7ad596eeff30~tplv-k3u1fbpfcp-watermark.image?)

### 运行工程
进去以后，先配置我们前面配置的Go，一路选择我们前面安装路径上的go即可：/usr/local/go

然后新建文件，运行

```go
package main

import (
	"log"
	"os"
	"os/exec"
	"syscall"
)

func main() {
	cmd := exec.Command("sh")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS,
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```

## 总结
本文介绍了如果使用JetBrains Gateway在远程服务器上进行开发，可以发现还挺好用的，这样的话，切换电脑设备之类的，不需要重新去安装一大推IDE了，直接装一个Gateway即可，工程和编程环境全部都保存在云端

缺点目前还没看出来，继续深度的使用看看
