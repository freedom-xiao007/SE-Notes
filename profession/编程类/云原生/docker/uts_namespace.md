# UTS Namespace
***

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

```sh
go run uts_namespace.go
```

```sh
pstree -pl
```

```sh
sh-4.2# hostname
VM-20-11-centos
sh-4.2# hostname -b test
sh-4.2# hostname
test
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