# Namespace
***

构建Linux容器的Namespace技术，它帮助进程隔离出自己单独的空间

```go
package main

import (
	"log"
	"os"
	"os/exec"
	"syscall"
)

func main() {
	cmd := exec.Command("sh")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS,
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```

### UTS Namespace
UTS Namespace主要用来隔离nodename和domainname两个系统标识。在UTSNamespace里面，每个Namespace允许有自己的hostname。

```sh
go run uts_namespace.go
```

```sh
pstree -pl
```

![屏幕截图 2022-03-02 061412.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8d13696a61fe4e79a46550d6cc7ab2ec~tplv-k3u1fbpfcp-watermark.image?)


```sh
sh-4.2# echo $$
20415
sh-4.2# readlink /proc/20415/ns/uts
uts:[4026532170]
sh-4.2# readlink /proc/20411/ns/uts
uts:[4026531838]
sh-4.2# readlink /proc/20376/ns/uts
uts:[4026531838]
sh-4.2# readlink /proc/20499/ns/uts
sh-4.2#
```

```sh
sh-4.2# hostname
VM-20-11-centos
sh-4.2# hostname -b test
sh-4.2# hostname
test
```

### IPC Namespace
IPC Namespace用来隔离System V IPC和POSIX message queues。每一个IPCNamespace都有自己的System V IPC和POSIX message queue。

```sh
➜  dockerDemo go run example/system/uts_namespace.go
sh-4.2# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages

sh-4.2# ipcmk -Q
Message queue id: 0
sh-4.2# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0xcf2d42dd 0          root       644        0            0

sh-4.2# exit


# 新开一个终端
➜  dockerDemo go run example/system/uts_namespace.go
sh-4.2# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages

sh-4.2# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages

sh-4.2# exit
```

### PID Namespace
PID Namespace是用来隔离进程ID的。同样一个进程在不同的PID Namespace里可以拥有不同的PID。这样就可以理解，在docker container 里面，使用ps-ef经常会发现，在容器内，前台运行的那个进程PID是1，但是在容器外，使用ps-ef会发现同样的进程却有不同的PID，这就是PID Namespace做的事情。

```sh
➜  dockerDemo go run example/system/uts_namespace.go
sh-4.2# echo $$
1
sh-4.2#

➜  dockerDemo pstree -pl
systemd(1)─┬─YDLive(1996)─┬─{YDLive}(1997)
           ├─sshd(6739)─┬─sshd(11283)───sftp-server(11357)
           │            ├─sshd(12353)───zsh(12371)───go(20255)─┬─uts_namespace(20291)─┬─sh
```

### Mount Namespace
Mount Namespace用来隔离各个进程看到的挂载点视图。在不同Namespace的进程中，看到的文件系统层次是不一样的。在Mount Namespace中调用mount（）和umount（）仅仅只会影响当前Namespace内的文件系统，而对全局的文件系统是没有影响的。

```sh
➜  dockerDemo go run example/system/uts_namespace.go
sh-4.2# ps -ef
Error, do this: mount -t proc proc /proc
sh-4.2# ls /proc/
1     12     1334   16     188    2017   208    257    27     28584  292  412  537   6739  754   buddyinfo  diskstats    iomem      kpagecount  mounts        self           timer_list
10    12353  1337   1667   18911  20255  21     25923  27573  286    35   46   5681  680   755   bus        dma          ioports    kpageflags  mtrr          slabinfo       timer_stats
1067  12371  1339   16684  19     20291  22     26     276    28715  36   47   6     682   756   cgroups    driver       irq        loadavg     net           softirqs       tty
11    13     14     16689  19296  20295  2200   261    27650  28750  37   48   630   7     759   cmdline    execdomains  kallsyms   locks       pagetypeinfo  stat           uptime
1132  13038  1467   16788  19301  2033   23     262    28     28754  377  49   633   724   8     consoles   fb           kcore      mdstat      partitions    swaps          version
1133  13056  15122  16804  1996   2039   24     263    28342  28778  38   50   6367  7496  824   cpuinfo    filesystems  keys       meminfo     sched_debug   sys            vmallocinfo
1155  1325   1555   17228  2      2040   24561  264    28503  29     4    51   646   752   9     crypto     fs           key-users  misc        schedstat     sysrq-trigger  vmstat
1163  1330   1590   18     20     2051   25     269    28570  291    404  528  65    753   acpi  devices    interrupts   kmsg       modules     scsi          sysvipc        zoneinfo
sh-4.2# mount -t proc proc /proc
sh-4.2# ls /proc
1          bus       cpuinfo    dma          filesystems  ioports   keys        kpageflags  meminfo  mtrr          sched_debug  slabinfo  sys            timer_stats  vmallocinfo
4          cgroups   crypto     driver       fs           irq       key-users   loadavg     misc     net           schedstat    softirqs  sysrq-trigger  tty          vmstat
acpi       cmdline   devices    execdomains  interrupts   kallsyms  kmsg        locks       modules  pagetypeinfo  scsi         stat      sysvipc        uptime       zoneinfo
buddyinfo  consoles  diskstats  fb           iomem        kcore     kpagecount  mdstat      mounts   partitions    self         swaps     timer_list     version
sh-4.2# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 04:11 pts/3    00:00:00 sh
root         5     1  0 04:12 pts/3    00:00:00 ps -ef
```

### User Namespace
User Namespace 主要是隔离用户的用户组ID。也就是说，一个进程的User ID 和Group ID在User Namespace内外可以是不同的。比较常用的是，在宿主机上以一个非root用户运行创建一个User Namespace，然后在User Namespace里面却映射成root 用户。这意味着，这个进程在User Namespace里面有root权限，但是在User Namespace外面却没有root的权限。从Linux Kernel 3.8开始，非root进程也可以创建User Namespace，并且此用户在Namespace里面可以被映射成root，且在Namespace内有root权限。

```sh
➜  dockerDemo uname -a
Linux VM-20-11-centos 3.10.0-1160.11.1.el7.x86_64 #1 SMP Fri Dec 18 16:34:56 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux

➜  dockerDemo go run example/system/uts_namespace.go
2022/03/03 04:28:33 fork/exec /usr/bin/sh: invalid argument
exit status 1

➜  dockerDemo ls /proc/13
ls: cannot read symbolic link /proc/13/exe: No such file or directory
attr       cgroup      comm             cwd      fd       io        map_files  mountinfo   net        oom_adj        pagemap      projid_map  schedstat  smaps  statm    task     wchan
autogroup  clear_refs  coredump_filter  environ  fdinfo   limits    maps       mounts      ns         oom_score      patch_state  root        sessionid  stack  status   timers
auxv       cmdline     cpuset           exe      gid_map  loginuid  mem        mountstats  numa_maps  oom_score_adj  personality  sched       setgroups  stat   syscall  uid_map

➜  dockerDemo echo 640 > /proc/sys/user/max_user_namespaces
➜  dockerDemo go run example/system/uts_namespace.go
2022/03/03 04:35:16 fork/exec /usr/bin/sh: operation not permitted
exit status 1
```

```go
// user namespace 需要额外设置的，但本机不行，需要注释掉
//cmd.SysProcAttr.Credential = &syscall.Credential{Uid: uint32(1), Gid: uint32(1)}
```

```sh
➜  dockerDemo id
uid=0(root) gid=0(root) groups=0(root)
➜  dockerDemo go run example/system/uts_namespace.go
sh-4.2$ id
uid=65534 gid=65534 groups=65534
```

### Network Namespace
Network Namespace 是用来隔离网络设备、IP地址端口等网络栈的Namespace。Network Namespace可以让每个容器拥有自己独立的（虚拟的）网络设备，而且容器内的应用可以绑定到自己的端口，每个Namespace内的端口都不会互相冲突。在宿主机上搭建网桥后，就能很方便地实现容器之间的通信，而且不同容器上的应用可以使用相同的端口。

```sh
➜  dockerDemo git:(master) ✗ ifconfig
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:3b:95:c2:6a  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.20.11  netmask 255.255.252.0  broadcast 10.0.23.255
        inet6 fe80::5054:ff:fe4f:72ca  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:4f:72:ca  txqueuelen 1000  (Ethernet)
        RX packets 4462637  bytes 5746186047 (5.3 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1143515  bytes 210183926 (200.4 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 243585  bytes 44119999 (42.0 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 243585  bytes 44119999 (42.0 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

➜  dockerDemo git:(master) ✗ go run example/system/uts_namespace.go
sh-4.2$ ifconfig
sh-4.2$
```

## 参考链接
- [error about User Namespace](https://github.com/xianlubird/mydocker/issues/3)
- [mydocker](https://github.com/xianlubird/mydocker)