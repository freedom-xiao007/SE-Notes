# 自制操作系统日记（5）：跳转到C语言执行
***

代码仓库地址：[https://github.com/freedom-xiao007/operating-system](https://github.com/freedom-xiao007/operating-system)

## 简介
在上篇中切换了CPU的64位模式，但后面是失败的，并没有真正切换，也没有相关的验证代码，本篇中终于修正并执行了C代码

## CPU模式切换修正
上篇中后面发现系统在Get SVGA时卡住了，并没有向下执行

后面搜索资料，最终尝试下来，感觉上使用qeme好像有些问题，同样的镜像在qemu中会不断循环重启，而在bochs中不会，所以我们重新抄了下代码，并且换了下虚拟机，换成bochs

文章的参考链接如下,写的很好，又学到了很多

- [【原创】计算机自制操作系统(Linux篇)一:用Nasm重写Linux引导启动程序](https://zhuanlan.zhihu.com/p/273251018)

我们直接抄它的三个文件：bootsect.asm、setup.asm、head.asm

在head.asm中，他是直接在里面模拟的c的main函数，我们将其注释掉，并修改call main为我们的c的start函数，具体修改如下：

修改call main 为 call _start; 在104行左右

注释掉所有的main函数代码，115到120行左右

```nasm
push 0 ;These are the parameters to main :-)
push 0 ;这些是调用main程序的参数（指init/main.c）。
push 0  
push L6 ;return address for main, if it decides to.
push _start ;'_main'是编译程序对main的内部表示方法。
jmp  setup_paging   ;这里用的JMP而不是call，就是为了在setup_paging结束后的
                    ;ret指令能去执行C程序的main() 
L6:
jmp L6 ;main程序绝对不应该返回到这里。不过为了以防万一，
     ;所以添加了该语句。这样我们就知道发生什么问题了。
     
     
     


; _main:      ;这里暂时模拟出C程序main() 
;      mov  esi,mainmsg                ;保护模式DS=0,数据用绝对地址访问
;      mov  cl, 0x09                   ;蓝色
;      mov  edi, 0xb8000+22*160        ;指定显示在某行,显卡内存地址需用绝对地址
;      call printnew                   ;0xb8000为字符模式下显卡映射到的内存地址 
;      ret  
```

这样对他的相关改造就OK了

## C代码植入
关于运行C代码的部分我们还是跟随《30天自制操作系统》的思路：将C代码转换成汇编码，和head.asm拼接到一起，然后直接编译即可

根据大体思路，我们的具体操作如下：

- 1.编写C代码
- 2.使用GCC将C代码生成.O文件
- 3.使用objconv将.O文件转成nasm汇编文件
- 4.使用python脚本处理调转换得到的汇编文件的一些不需要的地方
- 5.编写运行脚本，拼接文件，编译，运行

具体细节如下：

### 1.C代码编写
我们就简单起名叫start.c吧，里面就打印一个字符串，无限循环

```c
typedef unsigned char uint8_t;

void put_str(uint8_t* message);

void start(void) {
    put_str("0123456789");
    while(1);
}
```

put_str的函数目前我们用汇编进行实现，新增func.asm文件，编写put_str函数

```nasm
_put_str:
	mov  esi, esp ;保护模式DS=0,数据用绝对地址访问
	mov  cl, 0x09                   ;蓝色
	mov  edi, 0xb8000+22*160        ;指定显示在某行,显卡内存地址需用绝对地址
	call printnew                   ;0xb8000为字符模式下显卡映射到的内存地址 
	ret  
```

打印字符串就参考的head.asm里面的main函数

### 2.使用GCC将C代码生成.O文件
在Windows10上使用GCC需要安装一些东西

首先安装命令choco，管理员方式打开powershell，输入下面的命令：

```powershell
set-executionpolicy remotesigned
iwr https://chocolatey.org/install.ps1 -UseBasicParsing | iex
```

然后使用命令安装MinGW

```powershell
choco install mingw
```

安装完成后，重启启动下powershell就可以使用gcc命令，如下就是使用gcc将.c文件生成.o文件

```powershell
gcc -m32 -fno-asynchronous-unwind-tables -s -O2 -c -o .\\c\\start.o .\\c\\start.c
```

我们目前还是使用32位吧(命令的-m32)，64位有点把控不住......

### 3.使用objconv将.O文件转成nasm汇编文件
这个工具需要进行下载，下载地址为：https://www.agner.org/optimize/#objconv，在页面上找Object file converter，然后点击下载后对应安装即可

将.o文件转换成nasm文件命令如下：

```powershell
D:\\software\\objconv\\objconv.exe -fnasm .\\c\\start.o .\\c\\nasm\\start.asm
```

### 4.使用python脚本处理调转换得到的汇编文件的一些不需要的地方
生成的文件中有很多地方错误和不必要的代码，需要进行处理

如何人工去处理太慢了，这里使用编写python脚本进行处理

脚本大致如下,还是比较简单的，就是读文件，读取一行内容后进行过滤，如果符合条件，写入新文件中

```python
if __name__ == "__main__":
    with open("E:\\code\\other\\self\\operating-system\\c\\clean\\start.asm", "w") as fw:
        with open("E:\\code\\other\\self\\operating-system\\c\\nasm\\start.asm", "r") as fr:
            content = fr.readline()
            while content:
                if content.startswith("global") or content.startswith("extern"):
                    content = fr.readline()
                    continue
                content = content.replace("noexecute", "")
                content = content.replace("execute", "")
                fw.write(content)
                content = fr.readline()
```

在搜索的资料中，需要进行下面的操作：

- 使用global定义的函数名称标签(extern目前暂时去掉)
- 去掉.SECTION一行中的execute或noexecute，和（如果有需要的话）align=N语句
- 去掉default rel行(目前的文件中没有发现，暂时没有处理)

这样就大致处理完成了

### 5.编写运行脚本，拼接文件，编译，运行
下面就是最后的拼接和编译运行了，我们将所有的步骤放到bat脚本中，一键运行，脚本内容如下：

```powershell
# 编译
D:\\software\\NASM\\nasm.exe bootsect.asm -o bootsect.bin -l bootsect.lst
D:\\software\\NASM\\nasm.exe setup.asm -o setup.bin  -l setup.lst

# 将C代码转换后拼接编译
gcc -m32 -fno-asynchronous-unwind-tables -s -O2 -c -o .\\c\\start.o .\\c\\start.c
D:\\software\\objconv\\objconv.exe -fnasm .\\c\\start.o .\\c\\nasm\\start.asm
D:\\software\\python3\\python.exe E:\\code\\python\\self\\tools\\tools\\objconv2nasm_clearn.py
copy /B head.asm+.\\c\\clean\\start.asm+func.asm  kernel.asm
D:\\software\\NASM\\nasm.exe kernel.asm -o kernel.bin -l kernel.lst

# 将所有的文件整合成镜像
copy /B bootsect.bin+setup.bin+kernel.bin  os.iso

# 使用bochs运行镜像
D:\\software\\Bochs-2.7\\bochs -q -f D:\\software\\Bochs-2.7\\dlxlinux\\bochsrc_m.bxrc
```

bochs的安装和使用说明参考后面的，运行结果如下：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fdc99a612e2c441bac767e7797b168b1~tplv-k3u1fbpfcp-watermark.image?)

## bochs安装使用说明
首先bochs的下载地址为：https://sourceforge.net/projects/bochs/

解压后，我们需要修改：D:\software\Bochs-2.7\dlxlinux\bochsr.bxrc成D:\software\Bochs-2.7\dlxlinux\bochsr_m.bxrc

主要修改文件的对应路径和启动方式，整个配置文件如下：

```text
###############################################################
# bochsrc.txt file for DLX Linux disk image.
###############################################################

# how much memory the emulated machine will have
megs: 32

# filename of ROM images,替换原来的相对路径为绝对路径
romimage: file=D:\\software\\Bochs-2.7\\BIOS-bochs-latest
vgaromimage: file=D:\\software\\Bochs-2.7\\VGABIOS-lgpl-latest

# what disk images will be used 
# 指明启动的镜像为我们的os.iso镜像文件
floppya: 1_44=E:\\code\\other\\self\\operating-system\\os.iso, status=inserted
floppyb: 1_44=floppyb.img, status=inserted

# hard disk
ata0: enabled=1, ioaddr1=0x1f0, ioaddr2=0x3f0, irq=14
ata0-master: type=disk, path="D:\\software\\Bochs-2.7\\dlxlinux\\hd10meg.img", cylinders=306, heads=4, spt=17

# choose the boot disk.修改启动方式为软盘
boot: floppy

# default config interface is textconfig.
#config_interface: textconfig
#config_interface: wx

#display_library: x
# other choices: win32 sdl wx carbon amigaos beos macintosh nogui rfb term svga

# where do we send log messages?
log: bochsout.txt

# disable the mouse, since DLX is text only
mouse: enabled=0

# set up IPS value and clock sync
cpu: ips=15000000
clock: sync=both

# enable key mapping, using US layout as default.
#
# NOTE: In Bochs 1.4, keyboard mapping is only 100% implemented on X windows.
# However, the key mapping tables are used in the paste function, so 
# in the DLX Linux example I'm enabling keyboard_mapping so that paste 
# will work.  Cut&Paste is currently implemented on win32 and X windows only.

# 替换原来的相对路径为绝对路径
keyboard: keymap=D:\\software\\Bochs-2.7\\keymaps/x11-pc-us.map
#keyboard: keymap=D:\\software\\Bochs-2.7\\keymaps/x11-pc-fr.map
#keyboard: keymap=D:\\software\\Bochs-2.7\\keymaps/x11-pc-de.map
#keyboard: keymap=D:\\software\\Bochs-2.7\\keymaps/x11-pc-es.map
```

这样就OK了，这个是启动的配置文件

在我们的一键运行脚本中，配置使用该文件即可

```powershell
D:\\software\\Bochs-2.7\\bochs -q -f D:\\software\\Bochs-2.7\\dlxlinux\\bochsrc_m.bxrc
```

## 总结
最后的打印字符串还是有点问题，但最终的目的是达到了，调用C语言的代码（后面也查了资料，但目前的水平短时间内解决不了，只能放到后面看看了）

## 参考链接
- [choco : 无法将“choco”项识别为 cmdlet、函数、脚本文件或可运行程序的名称。请检查名称的拼写，如果包括路径，请确保路径正 确，然后再试一次。](https://blog.csdn.net/weixin_45954124/article/details/111314929)
- [MinGW-w64 12.2.0](https://community.chocolatey.org/packages/mingw)
- [Object file converter download](https://www.agner.org/optimize/#objconv)
- [bochs download](https://sourceforge.net/projects/bochs/)
- [用bochs调试自己写的系统引导代码](https://www.cnblogs.com/chengxuyuancc/archive/2013/05/13/3076524.html)
- [gcc生成32位的nasm汇编代码](https://visualgmq.gitee.io/2020/06/06/gcc%E7%94%9F%E6%88%9032%E4%BD%8D%E7%9A%84nasm%E6%B1%87%E7%BC%96%E4%BB%A3%E7%A0%81/)