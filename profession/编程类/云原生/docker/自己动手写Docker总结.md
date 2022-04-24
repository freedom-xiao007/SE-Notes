# 自己动手写 Docker 系列

---

一起养成写作习惯！这是我参与「掘金日新计划 · 4 月更文挑战」的第 25 天，[点击查看活动详情](https://juejin.cn/post/7080800226365145118)。

## 简介

本文章系列起源于：《自己动手写 Docker》,编程实践方得真知，虽然大部分代码书中都有，但还是遇到了不少困难，下面是对于自己写的 Docker Demo 的总览

## 工程说明

同时放到了 Gitee 和 Github 上，都可进行获取

- [Gitee: https://gitee.com/free-love/docker-demo](https://gitee.com/free-love/docker-demo)
- [GitHub: https://github.com/lw1243925457/dockerDemo](https://github.com/lw1243925457/dockerDemo)

工程基于 Go：1.17

由于系统原因，不能在 Windows 平台运行，只能在 Linux 平台上运行

本工程实现了的大致功能清单如下：

- [x] 构造实现 run 命令版本的容器
  - [x] 实现 run 命令
  - [x] 增加容器限制
  - [x] 增加管道及环境变量识别
- [x] 使用 busybox 创建容器
  - [x] 使用 AUFS 包装 busybox
  - [x] 实现 volume 数据卷
  - [x] 实现简单镜像打包
- [x] 实现容器的后台运行
- [x] 实现查看运行中的容器
- [x] 实现查看容器日志
- [x] 实现进入容器 Namespace
- [x] 实现停止容器
- [x] 实现删除容器
- [x] 实现通过容器制作镜像
- [x] 实现容器指定环境变量运行
- [x] 容器地址分配
- [x] 创建 Bridge 网络
- [x] 在 Bridge 网络创建容器

## 环境说明

本文基于下面的环境进行开发发运：

- Ubuntu 20 TLS ：本地搭建的 Ubuntu 系统
- Centos7 ：腾讯云服务器，也能跑本工程下的所有代码

注：Windows 不能运行该工程，因为其中有些库是 Linux 采用的，

但如果想要写 Windows 的话，可以仓库 RunC 中关于 Windows 相关的代码

如果后期有时间的话，本工程也尝试适配下 Windows 系统

看了下，windows 是基于：[https://github.com/microsoft/hcsshim](https://github.com/microsoft/hcsshim)

感觉难道有点大，看看后面时间了，时间不紧的话，可以尝试尝试

## 运行说明

docker demo 的代码都位于文件夹：mydocker 下

可参考下面的方式运行：

```shell
go mod init dockerDemo
go mod tidy
go build mydocker/main.go
./main run -ti /bin/sh
```

需求安装 Go 环境：https://go.dev/doc/install

根据官网教程进行安装即可

工程运行需要基础镜像：busybox，在文章中有说如何进行配置安装：[自己动手写 Docker 系列 -- 4.1 使用 busybox 创建容器](https://juejin.cn/post/7082480992614613022)

本文为了图便宜犯了一个错误，将容器的挂载数据卷设置成了工程运行时的所在目录，导致不同环境运行会有些问题，这是个教训

如果在克隆本工程，运行时出现报错：找不到 /proc

需要将 busybox 下的内容复制到工程根目录下的 /mnt 目录下

## 实践文档

在编写代码时，也遇到过不少问题，基本上都对过程进行记录，代码相对书中应该是比较完整，如果在编写过程中遇到问题，可以当做相关的参考

- [自己动手写 Docker 系列 -- 3.1 构造实现 run 命令版本的容器](https://juejin.cn/post/7081379481910411294)
- [自己动手写 Docker 系列 -- 3.2 增加容器资源限制](https://juejin.cn/post/7081757532053569543)
- [自己动手写 Docker 系列 -- 3.3 使用命令管道优化参数传递](https://juejin.cn/post/7082082864098967565)
- [自己动手写 Docker 系列 -- 4.1 使用 busybox 创建容器](https://juejin.cn/post/7082480992614613022)
- [自己动手写 Docker 系列 -- 4.2 使用 AUFS 包装 busybox](https://juejin.cn/post/7082873999872491527)
- [自己动手写 Docker 系列 -- 4.3 实现 volume 数据卷](https://juejin.cn/post/7083203141440634916)
- [自己动手写 Docker 系列 -- 5.1 实现容器的后台运行](https://juejin.cn/post/7083606684358148103)
- [自己动手写 Docker 系列 -- 5.2 实现查看运行中的容器](https://juejin.cn/post/7083966324442923015)
- [自己动手写 Docker 系列 -- 5.3 实现 logs 命令查看容器日志](https://juejin.cn/post/7084371162905444382)
- [自己动手写 Docker 系列 -- 5.4 实现进入容器的 namespace，exec 命令](https://juejin.cn/post/7084729876522991653)
- [自己动手写 Docker 系列 -- 5.5 实现容器停止](https://juejin.cn/post/7085077429412167693)
- [自己动手写 Docker 系列 -- 5.6 实现删除容器](https://juejin.cn/post/7085465652336525320)
- [自己动手写 Docker 系列 -- 5.7 实现通过容器制作镜像](https://juejin.cn/post/7086069688664326157)
- [自己动手写 Docker 系列 -- 5.8 实现容器制定环境变量运行](https://juejin.cn/post/7086220954551975973)
- [自己动手写 Docker 系列 -- 6.1 ip 分配管理](https://juejin.cn/post/7086559244275122207)
- [自己动手写 Docker 系列 -- 6.2 创建网络](https://juejin.cn/post/7087038556614426654)
- [自己动手写 Docker 系列 -- 6.3 手动配置容器网络(上)](https://juejin.cn/post/7089679899392376868/)
- [自己动手写 Docker 系列 -- 6.3 手动配置容器网络(下)](https://juejin.cn/post/7089927227894136846)
- [自己动手写 Docker 系列 -- 6.5 启动时给容器配置网络](https://juejin.cn/post/7090259129985400846)

## 参考资料

- 《自己动手写 Docker》：非常好的书籍，值得一看并实操
