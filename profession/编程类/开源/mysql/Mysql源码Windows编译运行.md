# Mysql源码阅读 -- Windows10编译运行MySQL源码

持续创作，加速成长！这是我参与「掘金日新计划 · 6 月更文挑战」的第3天，[点击查看活动详情](https://juejin.cn/post/7099702781094674468?utm_source=xitongxiaoxi&utm_medium=push&utm_campaign=kechengfenxiao)


## 简介

看了一些MySQL相关的书籍和文章，但感觉知识还不是自己的，打算看一看源码，本篇文章就从Windows10下编译运行MySQL源码开始



## 源码准备

mysql是一个开源项目，github地址如下，可以直接进行克隆

- [mysql-server](https://github.com/mysql/mysql-server)

下载完成后，我们使用github desktop打开后切换到5.7版本分支（或者使用命令行切换也行）


![屏幕截图 2022-06-18 082236.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/03ad8926ae1d4751934cffe789d00470~tplv-k3u1fbpfcp-watermark.image?)



## 资源准备

众所周知，完事开头难，程序员的世界，被环境折磨相信很多人感同身受，所以下面就相信列举博主在过程中使用到的资源和下载链接



机器环境：Windows10，下面是需要下载的相关东西



- 下载后放到资源目录下：[boost_1_59_0.zip](http://sourceforge.net/projects/boost/files/boost/1.59.0/boost_1_59_0.zip)
- 下载后点击进行安装：[bison-2.4.1-setup.exe](https://sourceforge.net/projects/gnuwin32/files/bison/2.4.1/bison-2.4.1-setup.exe/download)
- 下载后点击进行安装，记得确认添加到PATH：[cmake-3.17.2-win64-x64.msi - GitHub](https://github.com/Kitware/CMake/releases/download/v3.17.2/cmake-3.17.2-win64-x64.msi)
- 下载后解压放到资源目录下：[boost_1_59_0.tar.gz](http://sourceforge.net/projects/boost/files/boost/1.59.0/boost_1_59_0.tar.gz)
- Visual Studio 2019，社区版即可（2022应该也可以，可以试试）：https://visualstudio.microsoft.com/zh-hans/downloads/



## 编译Sln

我们先在源码根目录下创建两个文件夹：boot和release

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/54dbdb786bab4d66a8a05fdc3eff79db~tplv-k3u1fbpfcp-zoom-1.image)

然后把boost_1_59_0.tar.gz压缩包放到里面去，并且解压boost_1_59_0.tar.gz


![屏幕截图 2022-06-18 081705.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb6368e170b141f0a43cb83fb1f9c8cf~tplv-k3u1fbpfcp-watermark.image?)



然后进入新建后的release文件夹，运行下面的命令

```shell
cd D:\Code\c++\self\mysql-server\release
cmake ..  -DDOWNLOAD_BOOST=1 -DWITH_BOOST="D:\Code\c++\self\mysql-server\boost\boost_1_59_0.tar.gz"
```



成功后，我们就可以看到slm文件了：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d793050541114dfb9d5b7249e757325e~tplv-k3u1fbpfcp-zoom-1.image)



## 运行

双击上面生成的MySQL.sln文件，visual studio随着就打开了，下面我们需要配置一下



1.打开MySQL的调试模式，文件路径如下：D:\Code\c++\self\mysql-server\sql\sql_locale.cc，修改如下


![屏幕截图 2022-06-18 082702.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fc441499743149f8a287835844ca71a5~tplv-k3u1fbpfcp-watermark.image?)



2.增加启动参数：--console --initialize

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/412dc35c85594030aa57f215b8f71d53~tplv-k3u1fbpfcp-zoom-1.image)



3.将根目录下的：sql\sql_locale.cc,用 [utf-8 + BOM] 格式保存一下(用记事本打开，保存之类的)



4.对着mysqld鼠标右键，设置为启动项目，然后debug方式运行


![屏幕截图 2022-06-18 083303.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/668b7d27084849a9bfbb258fe44d75e4~tplv-k3u1fbpfcp-watermark.image?)


![屏幕截图 2022-06-18 083345.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/35cb508589ed485fa28f4c9d8a619555~tplv-k3u1fbpfcp-watermark.image?)

运行后，我们看到了初始的密码，记得把密码保存下来

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/035eaa0da0a54bee85a23276e0047187~tplv-k3u1fbpfcp-zoom-1.image)



5.再次修改参数，把第二步的后面的--initialize去掉，然后启动，嘿嘿，成了


![屏幕截图 2022-06-18 083544.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/879f4a7ad8ad4bdaad3e754c09cc1a30~tplv-k3u1fbpfcp-watermark.image?)





## 总结

本篇算是一篇开胃菜吧，把源码阅读的基础准备好，等待后序工作了



## 参考链接

- [第二站使用visual studio 对mysql进行源码级调试 - 博客园](https://www.cnblogs.com/huangxincheng/p/13084736.html)
- [Initializing mysql directory error](https://stackoverflow.com/questions/37644118/initializing-mysql-directory-error) error](https://stackoverflow.com/questions/37644118/initializing-mysql-directory-error)