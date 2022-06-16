# Windows10编译运行MySQL源码



## 简介

看了一些MySQL相关的书籍和文章，但感觉知识还不是自己的，打算看一看源码，本篇文章就从Windows10下编译运行MySQL源码开始



## 源码准备

mysql是一个开源项目，github地址如下，可以直接进行克隆

- [mysql-server](https://github.com/mysql/mysql-server)



## 资源准备

众所周知，完事开头难，程序员的世界，被环境折磨相信很多人感同身受，所以下面就相信列举博主在过程中使用到的资源和下载链接



机器环境：Windows10，下面是需要下载的相关东西



- 下载后放到资源目录下：[boost_1_59_0.zip](http://sourceforge.net/projects/boost/files/boost/1.59.0/boost_1_59_0.zip)
- 下载后点击进行安装：[bison-2.4.1-setup.exe](https://sourceforge.net/projects/gnuwin32/files/bison/2.4.1/bison-2.4.1-setup.exe/download)
- 下载后点击进行安装，记得确认添加到PATH：[cmake-3.17.2-win64-x64.msi - GitHub](https://github.com/Kitware/CMake/releases/download/v3.17.2/cmake-3.17.2-win64-x64.msi)
- 下载后解压放到资源目录下：[boost_1_59_0.tar.gz](http://sourceforge.net/projects/boost/files/boost/1.59.0/boost_1_59_0.tar.gz)
- Visual Studio 2019，社区版即可（2022应该也可以，可以试试）：https://visualstudio.microsoft.com/zh-hans/downloads/



## 编译Sln

```shell
cd D:\Code\c++\self\mysql-server\release
cmake ..  -DDOWNLOAD_BOOST=1 -DWITH_BOOST="D:\Code\c++\self\mysql-server\boost\boost_1_59_0.tar.gz"
```



## 运行



D:\Code\c++\self\mysql-server\sql\sql_locale.cc



SdeiuNoNf7-c



## 参考链接

- [第二站使用visual studio 对mysql进行源码级调试 - 博客园](https://www.cnblogs.com/huangxincheng/p/13084736.html)
- [Initializing mysql directory error](https://stackoverflow.com/questions/37644118/initializing-mysql-directory-error)