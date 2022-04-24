# 自己动手写 Docker 系列

---

## 文章系列

### 前置知识

需要了解和掌握下面的知识，最后自己动手实践，并记录成博客

- [ ] Linux Namespace
- [ ] Linux CGroups
- [ ] Union File System

### Docker 实现

- [ ] 构造实现 run 命令版本的容器
  - [ ] 实现 run 命令
  - [ ] 增加容器限制
  - [ ] 增加管道及环境变量识别
- [ ] 使用 busybox 创建容器

  - [ ] 使用 AUFS 包装 busybox
  - [ ] 实现 volume 数据卷
  - [ ] 实现简单镜像打包

- [ ] 实现容器的后台运行
- [ ] 实现查看运行中的容器
- [ ] 实现查看容器日志
- [ ] 实现进入容器 Namespace
- [ ] 实现停止容器
- [ ] 实现删除容器
- [ ] 实现通过容器制作镜像
- [ ] 实现容器指定环境变量运行

- [ ] 网络虚拟化技术
- [ ] 构建容器网络模型
- [ ] 容器地址分配
- [ ] 创建 Bridge 网络
- [ ] 在 Bridge 网络创建容器
- [ ] 容器跨主机网络

- [ ] nginx 使用验证
- [ ] flask+redis 使用验证

### 文章列表

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
- [自己动手写 Docker 系列 -- 6.3 手动配置容器网络(下)](https://juejin.cn/post/7089679899392376868/)
- [自己动手写 Docker 系列 -- 6.5 启动时给容器配置网络](https://juejin.cn/post/7090259129985400846)

## 参考资料

- 《自己动手写 Docker》
