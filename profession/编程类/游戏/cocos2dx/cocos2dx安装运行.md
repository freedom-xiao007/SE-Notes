# Cocos2dx 安装运行
***

持续创作，加速成长！这是我参与「掘金日新计划 · 10 月更文挑战」的第1天，[点击查看活动详情](https://juejin.cn/post/7147654075599978532?utm_source=creator&utm_medium=banner&utm_campaign=gengwen202210)

## 简介
最近在研究游戏，体验了下unity、unreal engine、cocos、jmonkey等游戏引擎，正好写文章记录下，本篇是cocos2dx的安装运行

## 环境及依赖安装说明
博主的电脑环境如下：

- window10
- [python2.7](https://www.python.org/download/releases/2.7/)：这个要注意，cocos2dx使用的是python2，不是python3，试了下，python3确实不行
- [cocos2dx-V3.17.1版本](https://www.cocos.com/cocos2dx):没有使用最新版的4，看更新没有啥大操作，而且试了下好像和文档不太同步？新建js工程一直报错，尝试好久没有解决，最终用老版本3顺利运行，所以还是用3吧
- [Visual Studio IDE 2013/2015/2017](https://visualstudio.microsoft.com/zh-hans/thank-you-downloading-visual-studio/?sku=Community&rel=15)：不能用高版本，会报错了，但微软的官方下载页面让人摸不着头脑，最终还是找到办法下载了老版本,在Stack Overflow上找到了方法，最后的rel修改版本，虽然不是预想中的15版本，是17，但能用就行了
- Android SDK：发布Android版本的时候需要，可以暂时不配置，目前不是必要的

用两个需要我们手动加入环境变量，python2直接用绝对路径也行,环境变量配置后记得注销电脑后再登录，这样就生效了

## 运行
安装完成后，进入cocos的根目录，运行下面的命令进行配置下：

```sh
python setup.py
```

然后我们就在cocos的根目录下的：tools\cocos2d-console\bin，有对应的cocos命令，将这个bin目录加入环境变量后，注销重启生效

重启后，我们就可以直接使用cocos命令：

```sh
PS E:\code\js\self> cocos -v
cocos2d-x-3.17.2
Cocos Console 2.3
```

接下来我们使用命令创建一个工程：

```sh
PS E:\code\js\self> cocos new MyGame -p com.MyCompany.MyGame -l js -d ./MyCompany
> 拷贝模板到 E:\code\js\self\MyCompany\MyGame
> 拷贝引擎中的文件夹...
> 拷贝模板中的文件夹...
> 拷贝 cocos2d-x ...
> 替换文件名中的工程名称，'HelloJavascript' 替换为 'MyGame'。
> 替换文件中的工程名称，'HelloJavascript' 替换为 'MyGame'。
> 替换工程的包名，'org.cocos2dx.hellojavascript' 替换为 'com.MyCompany.MyGame'。
> 替换 Mac 工程的 Bundle ID，'org.cocos2dx.hellojavascript' 替换为 'com.MyCompany.MyGame'。
> 替换 iOS 工程的 Bundle ID，'org.cocos2dx.hellojavascript' 替换为 'com.MyCompany.MyGame'。
```

指定语言为js，其他选项有lua和c++

然后可以运行测试，使用web方式：

```sh
# 首先进入我们的游戏根目录
PS E:\code\js\self\MyCompany\MyGame> cocos run . -p web
编译模式：debug
部署模式：debug
启动应用。
尝试启动服务器 127.0.0.1:8000
HTTP 服务已启动，主机：127.0.0.1，端口：8000 ...
```

这样就自动打开浏览器，显示初始的示例了

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0823cc6e67e54d31ae0b6b1a831f7148~tplv-k3u1fbpfcp-watermark.image?)

## 总结
到这就基本运行起来了，其他的如发布之类的，官方文档里面都有

## 参考链接
- [How to download Visual Studio Community Edition 2017 (not 2019)](https://stackoverflow.com/questions/55837625/how-to-download-visual-studio-community-edition-2017-not-2019)
- [cocos 命令](https://docs.cocos.com/cocos2d-x/manual/zh/editors_and_tools/cocosCLTool.html)