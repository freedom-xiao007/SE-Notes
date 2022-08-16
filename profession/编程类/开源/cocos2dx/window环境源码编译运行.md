# Cocos2dx windows 环境编译运行
***

## 简介
本文将详细介绍基于cocos2dx源码，在Windows环境中编译运行

## 编译运行

```powershell
set COCOS2DX_ROOT_PATH="D:\Code\C++\self\cocos2d-x"
```

nupengl nupengl.core
glfw

```text
正在尝试收集与目标为“native,Version=v0.0”的项目“cpp-tests”有关的包“glfw.3.3.8”的依赖项信息
收集依赖项信息花费时间 1.7 sec
正在尝试解析程序包“glfw.3.3.8”的依赖项，DependencyBehavior 为“Lowest”
解析依赖项信息花费时间 0 ms
正在解析操作以安装程序包“glfw.3.3.8”
已解析操作以安装程序包“glfw.3.3.8”
  GET https://api.nuget.org/v3-flatcontainer/glfw/3.3.8/glfw.3.3.8.nupkg
  OK https://api.nuget.org/v3-flatcontainer/glfw/3.3.8/glfw.3.3.8.nupkg 192 毫秒
已通过内容哈希 XWwZh6BByUqFF5ZYQVG/1ipBEpjy01pnDONQOHD82Tm0nedhzWvhRqrKVvpB6v7/5pLR7f35sJxqYmo+C9wJTQ== 从 https://api.nuget.org/v3/index.json 安装 glfw 3.3.8 。
正在将程序包“glfw.3.3.8”添加到文件夹“D:\Code\C++\self\cocos2d-x\tests\cpp-tests\win32-build\packages”
已将程序包“glfw.3.3.8”添加到文件夹“D:\Code\C++\self\cocos2d-x\tests\cpp-tests\win32-build\packages”
已将程序包“glfw.3.3.8”添加到“packages.config”
已将“glfw 3.3.8”成功安装到 cpp-tests
执行 nuget 操作花费时间 20.79 sec
已用时间: 00:00:22.7680351
========== 已完成 ==========
```

## 参考链接
- [VS2019上配置OPENGL包GLFW](https://codeantenna.com/a/ckdCYIiYI5)
- [The OpenGL Extension Wrangler Library download](http://glew.sourceforge.net/)