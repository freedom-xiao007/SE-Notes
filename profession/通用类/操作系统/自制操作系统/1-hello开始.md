# 自制操作系统日记（一）：显示hello world开始旅程
***

## 简介
最近看了不少底层方面的东西，但还是得动手才能真正掌握，感觉操作系统也能整整了，于是就有了这系列，惯例的以hello开始

## 准备工作
本文操作的主要是下面的资料来源：

- 《30天自制操作系统》：大部头，读完真花不少时间
- 《极客时间：操作系统实战45讲》

资料中有些工具不太好弄，所以会和他们有些区别，首先我们安装下两个软件：

- [NASM](https://www.nasm.us/):汇编语言编译器，第一本书是用自写的nask，但找不到啊，只能网上搜索，然后用这个了（可能后面会导致一些困难，但也没办法了，总有困难在前方）
- [qemu](https://qemu.weilnetz.de/w64/)：虚拟机，用来启动我们的操作系统，vm和物理机感觉太麻烦了，这个直接用一行命令启动就行了，很方便

下载链接对应的点击跳转即可，下载完成后，一路点点确认即可，这个应该轻车熟路了吧

博主的系统是window10，不同系统的注意软件的适配下载

## hello编写
我们开始新建一个工程文件夹，博主的是operating-system，你们随意

然后新建一个myOS.asm文件，输入下面的代码（先原本照抄再说，虽然以前在学校学过，但现在基本忘了）

```asm
; cherry-os
ORG 0x7c00 ;指定程序装载的位置

;下面用于描述FAT12格式的软盘
JMP entry
DB 0x90
DB "CHRRYIPL" ;启动区的名称可以是任意的字符串，但长度必须是8字节
DW 512; 每一个扇区的大小，必须是512字节
DB 1 ;簇的大小（必须为1个扇区)
DW 1 ;FAT的起始位置（一般从第一个扇区开始）
DB 2 ;FAT的个数 必须是2
DW 224;根目录的大小 一般是224项
DW 2880; 该磁盘的大小 必须是2880扇区
DB 0xf0;磁盘的种类 必须是0xf0
DW 9;FAT的长度 必须是9扇区
DW 18;1个磁道(track) 有几个扇区 必须是18
DW 2; 磁头个数 必须是2
DD 0; 不使用分区，必须是0
DD 2880; 重写一次磁盘大小
DB 0,0,0x29 ;扩展引导标记 固定0x29
DD 0xffffffff ;卷列序号
DB "CHERRY-OS  " ;磁盘的名称（11个字节）
DB "FAT12   " ;磁盘的格式名称（8字节）
TIMES 18 DB 0; 先空出18字节 这里与原文写法不同

;程序核心
entry:
    MOV AX,0  ;初始化寄存器
    MOV SS,AX
    MOV SP,0x7c00
    MOV DS,AX
    MOV ES,AX
    MOV SI,msg
putloop:
    MOV AL,[SI]
    ADD SI,1
    CMP AL,0
    JE fin
    MOV AH,0x0e ;显示一个文字
    MOV BX,15 ;指定字符的颜色
    INT 0x10 ;调用显卡BIOS
    JMP putloop
fin:
    HLT ;CPU停止,等待指令
    JMP fin ;无限循环
msg:
    DB 0x0a , 0x0a ;换行两次
    DB "hello, my OS"
    DB 0x0a
    DB 0
    
    TIMES 0x1fe-($-$$) DB 0 ;填写0x00,直到0x001fe
    
    DB 0x55, 0xaa
```

## 编译运行
正常情况下，我们需要把上面安装的两个软件加入环境变量中，但简单点，直接使用绝对路径搞定它（注销用户麻烦）

先进入我们的工程目录，启动powershell或者cmd都行

使用nasm命令，将文件编程img，下面是博主本地的安装路径

```shell
D:\software\NASM\nasm.exe .\myOS.asm -o .\myOS.img
```

运行完成后，会在工程目录下生成myOS.img文件

接下来，使用qemu运行我们的镜像文件

```shell
 D:\software\qemu\qemu-system-i386.exe .\myOS.img
```

可以看到下面的启动窗口，嘿嘿嘿，成功的走出第一步

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3d88f1928737448b998b27b9c766406c~tplv-k3u1fbpfcp-watermark.image?)

## 参考链接
- [自制操作系统-使用汇编显示 hello world](https://blog.csdn.net/weixin_30307267/article/details/97345011)