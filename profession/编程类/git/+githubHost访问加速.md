# Github Host 访问加速
***
## 简介
在国内众所周知的访问git的速度贼慢，有时候图片也加载不出来，本文就介绍如何通过配置host文件，加速访问GitHub

## 配置Host加速访问
### 获取Host
访问下面的链接，下载最新的Host，楼主是个好人

- [https://cdn.jsdelivr.net/gh/521xueweihan/GitHub520@main/hosts 作者：Acwinuxos系统 https://www.bilibili.com/read/cv10607526 出处：bilibili](https://cdn.jsdelivr.net/gh/521xueweihan/GitHub520@main/hosts)

### 配置Host
将内容放到host文件的最后面，对应的系统的host文件位置如下，记得使用管理员权限：

- Windows 系统：C:\Windows\System32\drivers\etc\hosts
- Linux 系统：/etc/hosts
- Mac（苹果电脑）系统：/etc/hosts
- Android（安卓）系统：/system/etc/hosts
- iPhone（iOS）系统：/etc/hosts

默认配置完就生效的，如果没有生效，运行下面的命令：

- Windows：在 CMD 窗口输入：ipconfig /flushdns
- Linux 命令：sudo rcnscd restart
- Mac 命令：sudo killall -HUP mDNSResponder

## 参考链接
- [github直连访问方法，长期更新的github hosts](https://www.bilibili.com/read/cv10607526)