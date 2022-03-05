# Linux Cgroup
***

```go
package main

import (
	"fmt"
	"io/ioutil"
	"log"
	"os"
	"os/exec"
	"path"
	"strconv"
	"syscall"
)

// 挂载了memory subsystem的hierarchy的跟目录位置
const cgroupMemoryHierarchyMount = "/sys/fs/cgroup/memory"

func main() {
	if os.Args[0] == "/proc/self/exe" {
		// 容器进程
		fmt.Printf("current pid %d\n", syscall.Getpid())
		cmd := exec.Command("sh", "-c", `stress --vm-bytes 200m --vm-keep -m 1`)
		cmd.SysProcAttr = &syscall.SysProcAttr{}
		cmd.Stdin = os.Stdin
		cmd.Stdout = os.Stdout
		cmd.Stderr = os.Stderr
		if err := cmd.Run(); err != nil {
			fmt.Println(err)
			os.Exit(1)
		}
	}

	cmd := exec.Command("/proc/self/exe")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS |
			syscall.CLONE_NEWUSER | syscall.CLONE_NEWNET,
	}
	// user namespace 需要额外设置的，但本机不行，需要注释掉
	//cmd.SysProcAttr.Credential = &syscall.Credential{Uid: uint32(1), Gid: uint32(1)}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	// 注意是cmd.Start(),Run()会阻塞住
	if err := cmd.Start(); err != nil {
		fmt.Println("ERROR", err)
		os.Exit(-1)
	} else {
		// 得到fork出来进程映射在外部命名空间的pid
		fmt.Printf("%v", cmd.Process.Pid)
		// 在系统默认创建挂载了memory subsystem的Hierarchy上创建cgroup
		cgroupName := "testmemorylimit"
		_ = os.Mkdir(path.Join(cgroupMemoryHierarchyMount, cgroupName), 0755)
		// 将容器加入这个cgroup
		_ = ioutil.WriteFile(path.Join(cgroupMemoryHierarchyMount, cgroupName, "tasks"), []byte(strconv.Itoa(cmd.Process.Pid)), 0644)
		// 限制cgroup进程的使用
		_ = ioutil.WriteFile(path.Join(cgroupMemoryHierarchyMount, cgroupName, "memory.limit_in_bytes"), []byte("100m"), 0644)
	}
	if _, err := cmd.Process.Wait(); err != nil {
		log.Fatal(err)
	}
}
```

```sh
 uname -a
Linux 10-9-72-82 4.18.0-240.1.1.el8_3.x86_64 #1 SMP Thu Nov 19 17:20:08 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux

➜  ~ uname -a
Linux VM-20-11-centos 3.10.0-1160.11.1.el7.x86_64 #1 SMP Fri Dec 18 16:34:56 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
➜  ~

stress --vm-bytes 200m --vm-keep -m 1

cd /sys/fs/cgroup/memory
 mkdir test-limit-memory && cd test-limit-memory
 cat memory.limit_in_bytes
  echo 18201 >> tasks
  echo "100m" >> memory.limit_in_bytes
   echo 0 >> memory.swappiness
```

经过多次尝试没有达到书中的效果：

如果是先启动stress命令，然后再修改内存限制或将其pid加入tasks，多次尝试后，发现其内存不会降低

但如果是先将当前shell的pid加入tasks来现在100M的内容，然后启动stress名，但其直接启动失败，如下：

```sh
ecd\liuwei_1689727483221@vflb0qf5pdzfnun:/sys/fs/cgroup/memory/test-limit-memory$ stress --vm-bytes 200m --vm-keep -m 1
stress: info: [56947] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd
stress: FAIL: [56947] (415) <-- worker 56948 got signal 9
stress: WARN: [56947] (417) now reaping child worker processes
stress: FAIL: [56947] (451) failed run completed in 1s
```

感觉限制应该是生效了，但没有像书中那样看到明显的效果

如果各位老哥有什么更好的思路，请在评论区告知，非常感谢