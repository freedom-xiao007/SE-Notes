# minetest Window编译运行
***
## 简介
minetest是在GitHub开源的，使用C++编写的沙盒游戏：我的世界，一直以来对于该游戏的编写很是好奇，但在以前没有找到相关的源码（以前水平太菜了），今天逛GitHub的时候，发现这么一个项目，非常的感兴趣，于是想研究下。最开始肯定是本地运行了，博主的操作系统是Windows10

## 编译运行
### 相关的工具下载安装
根据官网中的编译指南：[GitHub README 中Windows编译部分](https://github.com/minetest/minetest)和[YouTube上的编译教学视频](https://www.youtube.com/watch?v=B4QnlJozFoM),需要下载安装下面的工具，具体请查看视频，对新手还是比较友好了

注：每个人的环境可能稍有不同，比如我就遇到了很多视频中没有遇到的问题，大部分都可以通过阅读官方文档解决，其他我遇到的在下面都有记录

- [Visual Studio 2015 or newer](https://visualstudio.microsoft.com)
- [CMake](https://cmake.org/download/)
- [vcpkg](https://github.com/Microsoft/vcpkg)
- [Git](https://git-scm.com/downloads)

### vcpkg
执行下面的命令，国内的环境下面会很慢，如果遇到下载不了的，只能手动到网上去搜索下载

温馨提示：一定要将其放到C盘下，然后执行相关的编译命令，博主放到D盘死活编译不过，放到C盘就继续编译下去了，离谱！

该步骤初次博主花了1个小时左右，才完成了，各位老哥记得放C盘下啊！

```shell script
git clone https://github.com/microsoft/vcpkg.git
cd vcpkg
./vcpkg install zlib curl[winssl] openal-soft libvorbis libogg sqlite3 freetype luajit gmp jsoncpp --triplet x64-windows
```

### Cmake
在如视频中使用cmake gui的时候，遇到了下面的问题：

```
Please add a manifest, or disable manifests by turning off
  VCPKG_MANIFEST_MODE.
```

这个错误的解决方式就是把：VCPKG_MANIFEST_MODE 勾选去掉

还有下面一个错误：

```
CMake Error at CMakeLists.txt:78 (message):
  IrrlichtMt is required to build the client, but it was not found.

  The Minetest team has forked Irrlicht to make their own customizations.  It
  can be found here: https://github.com/minetest/irrlicht
```

下面就一直报这个错，终止通过看CmakeLists.txt发送可以通过另外的方式搞这个，目前博主是通过这种方式编译通过的：

克隆：https://github.com/minetest/irrlicht, 到工程目录下，博主的是 D:\Code\C++\self\minetest\lib

改名为：irrlichtmt

Configuration 两次

REQUIRE_LUAJIT 选中

generate 一次

到这里终于编译成功了

### Visual Studio 2019 编译运行
在运行的过程中也遇到了问题：GL/xx.h文件找不到

解决的方案是从 OpenGL中点击各个头文件进去，下载复制，然后自己生成相关的文件：[https://www.khronos.org/registry/OpenGL/index_gl.php](https://www.khronos.org/registry/OpenGL/index_gl.php)

最后放到VS的相关目录下，我的是：D:\SoftWare\VisualStudio\IDE\VC\Tools\MSVC\14.16.27023\include\GL

注：目录14.xx.xxx我有两个，不确定是那个，我就所有的都放了

如视频中的，使用IDE打开工程解决方案：D:\Code\C++\self\minetest\build\ALL_BUILD.vcxproj

选择release方式，x64平台

all build

然后在项目跟目录下：D:\Code\C++\self\minetest\bin\Release\minetest.exe

点击后完美运行！

## 参考链接
- [https://www.khronos.org/registry/OpenGL/index_gl.php](https://www.khronos.org/registry/OpenGL/index_gl.php)
- [Setup OpenGL with Visual Studio 2017 on Windows 10](https://www.absingh.com/opengl/)