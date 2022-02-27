# Android 模拟器连接异常：Unable to connect to ADB server
***

## 简介
在使用Android Studio开始的过程是，偶尔会突然出现一只在检测设备，导致无法进行运行的调试的情况，本文记录下解决方法

## 现象描述
在启动Android Studio后，在设备状态栏中一直显示检测设备，没有版本点击运行和调试按钮

并且在日志提示中一直在显示下面的：

```text
远程主机强迫关闭了一个现有的链接

* daemon not running; starting now at tcp:5037

* daemon started successfully
```

按照网上大多数的解决办法，去杀死任务管理器中的adb进程，发现并没有，完全没有效果

而且按照上面的日志，感觉abd一直不断地重启之中

折腾了一个小时左右，使用下面的方法得到了解决

## 解决方法
- 1.首先关闭Android Studio

- 2.查看任务管理器，看看有没有adb的进程在运行（点击排序后，看看首字母A开头的即可），有则将其杀掉

- 3.进入adb工具目录，根据自己Android SDK的安装路径来，比如博主的是：D:\Software\Android\SDK\platform-tools，还可以通过点击Android Studio的setting，搜索sdb进行查看，如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/27b0c8a2496440f79bcfdecb98ff13a7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAX-iQp18=,size_20,color_FFFFFF,t_70,g_se,x_16)

- 4.进入相应的目录后，运行下面的命令：

```sh
PS D:\Software\Android\SDK\platform-tools> .\adb.exe start-server
* daemon not running; starting now at tcp:5037
* daemon started successfully
```

如果输出和上面一致，那说明可以了，接着重启Android Studio后，发现恢复正常

## 参考链接
- [Unable to connect to ADB server](https://stackoverflow.com/questions/31282816/unable-to-connect-to-adb-server)
- [Android studio中关于真机调试时远程主机强迫关闭了一个现有连接的解决方法](https://blog.csdn.net/weiyongle1996/article/details/71267311)
- [How to solve Cannot reach ADB server, attempting to reconnect.](https://www.reddit.com/r/AndroidStudio/comments/syo3vx/how_to_solve_cannot_reach_adb_server_attempting/)