---
title: 一步步自己做个Docker之Go调用Namespace
author: Salamander
tags:
  - Docker
  - Linux
categories:
  - Docker
  - Golang
date: 2020-03-26 21:00:00
---
![docker logo](/images/docker-logo.png)

本文环境：
* OS：Ubuntu 18.04.4 LTS
* Golang版本：1.12.13

## Golang
Go语言是Google开发的一种静态类型、编译型的高级语言，它设计的蛮简单的，学过C的话，其实上手Go很快的，当然相比于C的话，Go有垃圾回收和并发支持，所以写起来心智负担更低一点。  
对于Go的安装和配置，我以前写过一篇文章——[go语言基本配置](https://segmentfault.com/a/1190000008487280)，我这里就不在赘述了。Go1.11增加了`go modules`，使用它的话，就没必要一定要把代码放到`GOPATH`下面啦~\(≧▽≦)/~。 `go modules`详细
使用请参考[go mod 使用](https://juejin.im/post/5c8e503a6fb9a070d878184a)。  

<!-- more -->

## Go调用Namespace
其实对于Namespace这种系统调用，使用C语言描述是最好的（[上一篇文章](/2019/11/28/docker-Linux-Namespace-intro/)就是用C写的示例），但是C比较难，而且Docker也是用Go是实现的，所以我后面的文章都会用Go来写示例代码。  
这里我先写了一个`UTS Namespace`的例子，`UTS Namespace`主要用来隔离`nodename`和`domainname`这两个系统标识：   
```
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
`exec.Command("sh")`是指定了fork出来的新进程内的初始命令，`cmd.SysProcAttr`这行就是设置了系统调用函数，Go帮我们封装了[clone()](http://man7.org/linux/man-pages/man2/clone.2.html)函数，`syscall.CLONE_NEWUTS`这个标识符标明创建一个`UTS Namespace`。  
`go build .`编译代码后，执行程序时我们会遇到错误**fork/exec /bin/sh: operation not permitted**，这是因为`clone()`函数需要`CAP_SYS_ADMIN`权限（这个[问题](https://www.v2ex.com/t/618961)我在v站上问过），解决方法是添加设置 `uid` 映射： 
```
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
		Cloneflags:
			syscall.CLONE_NEWUTS |
			syscall.CLONE_NEWUSER,
		UidMappings: []syscall.SysProcIDMap{
			{
				ContainerID: 0,
				HostID:      os.Getuid(),
				Size:        1,
			},
		},
		GidMappings: []syscall.SysProcIDMap{
			{
				ContainerID: 0,
				HostID:      os.Getgid(),
				Size:        1,
			},
		},
	}

	// set identify for this demo
	cmd.Env = []string{"PS1=-[namespace-process]-# "}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```
我增加了`CLONE_NEWUSER`标识，让新进程在`User Namespace`中变成root用户。  
```
$ ./uts-easy 
-[namespace-process]-# id
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)
-[namespace-process]-# hostname -b bird
-[namespace-process]-# hostname
bird
```
启动另一个shell，查看宿主机上`hostname`:   
```
$ hostname
salamander-PC
```
可以看到，外部的`hostname`并没有被内部的修改所影响，这里我们大致感受了下`UTS Namespace`的作用。

## 增加IPC Namespace
`IPC Namespace`用来隔离**System V IPC和POSIX message queues**。每一个`IPC Namespace`都有自己的**System V IPC**和**POSIX message queues**。我们稍微改动一下上面的代码。  
```
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
		Cloneflags:
			syscall.CLONE_NEWUTS |
			syscall.CLONE_NEWUSER |
			syscall.CLONE_NEWIPC,   // 增加IPC Namespace
		UidMappings: []syscall.SysProcIDMap{
			{
				ContainerID: 0,
				HostID:      os.Getuid(),
				Size:        1,
			},
		},
		GidMappings: []syscall.SysProcIDMap{
			{
				ContainerID: 0,
				HostID:      os.Getgid(),
				Size:        1,
			},
		},
	}

	// set identify for this demo
	cmd.Env = []string{"PS1=-[namespace-process]-# "}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```
新开一个shell，在宿主机上创建一个message queue:
```
$ ipcs -q

--------- 消息队列 -----------
键        msqid      拥有者  权限     已用字节数 消息      

$ ipcmk -Q
消息队列 id：0
$ ipcs -q

--------- 消息队列 -----------
键        msqid      拥有者  权限     已用字节数 消息      
0xc59399dd 0          salamander 644        0            0
```
运行我们自己的程序：
```
$ ./uts-easy 
-[namespace-process]-# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
```
可以看到，在新的Namespace中，看不到宿主机上创建的`message queue`,说明IPC是隔离的。


## 增加PID Namespace

**PID Namespace**是用来隔离进程ID的。我们自己进入Docker 容器的时候，就会发现里面的前台进程的PID为1，但是在容器外PID却不是1，这就是通过**PID Namespace**做到的。修改上述的代码，增加`CLONE_NEWPID`：
```
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
		Cloneflags:
			syscall.CLONE_NEWUTS |
			syscall.CLONE_NEWUSER |
			syscall.CLONE_NEWIPC |
			syscall.CLONE_NEWPID, // 增加PID Namespace
		UidMappings: []syscall.SysProcIDMap{
			{
				ContainerID: 0,
				HostID:      os.Getuid(),
				Size:        1,
			},
		},
		GidMappings: []syscall.SysProcIDMap{
			{
				ContainerID: 0,
				HostID:      os.Getgid(),
				Size:        1,
			},
		},
	}

	// set identify for this demo
	cmd.Env = []string{"PS1=-[namespace-process]-# "}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```
运行我们的程序：
```
 ./uts-easy 
-[namespace-process]-# echo $$
1
```
可以看到在新的`PID Namespace`中进程ID为1。
```
$ pstree -pl | grep easy
           |               |                       |-bash(10739)---uts-easy(10768)-+-sh(10773)
           |               |                       |                               |-{uts-easy}(10769)
           |               |                       |                               |-{uts-easy}(10770)
           |               |                       |                               |-{uts-easy}(10771)
           |               |                       |                               `-{uts-easy}(10772)
```
而我们在宿主机上可以看到它实际的PID（`uts-easy`这个进程）为**10768**。  
如果细心点，我们会发现，在我们的程序中使用`ps`，`top`这些命令出来的结果跟宿主机上是一样的，这是因为这些命令其实是去使用**/proc**这个文件夹的内容，这个就需要下面的`Mount Namespace`了。

## 增加Mount Namespace

**Mount Namespace**用来隔离各个进程看到的挂载点视图。在不同Namespace的进程中，看到的文件系统层次是不一样的。在**Mount Namespace**中调用`mount()`和`unmount()`只会影响当前Namespace内的文件系统，而对全局的文件系统是没有影响的。  
看到这里，也许会想到`chroot()`。它也能将某一个子目录变为根节点。但是，**Mount Namespace**不仅能实现这个功能，而且能以更加灵活和安全的方式实现。  
现在继续修改上述代码，增加`CLONE_NEWNS`（Mount Namespace是Linux实现的第一个Namespace类型，因为，它的系统调用参数是**NEWNS**，NS是New Namespace的缩写。当时人们没有意识到，以后还会有很多类型的Namespace加入Linux大家庭）。
```
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
		Cloneflags:
			syscall.CLONE_NEWUTS |
			syscall.CLONE_NEWUSER |
			syscall.CLONE_NEWIPC |
			syscall.CLONE_NEWPID|
			syscall.CLONE_NEWNS,  // 增加Mount Namespace
		UidMappings: []syscall.SysProcIDMap{
			{
				ContainerID: 0,
				HostID:      os.Getuid(),
				Size:        1,
			},
		},
		GidMappings: []syscall.SysProcIDMap{
			{
				ContainerID: 0,
				HostID:      os.Getgid(),
				Size:        1,
			},
		},
	}

	// set identify for this demo
	cmd.Env = []string{"PS1=-[namespace-process]-# "}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```
运行程序，查看/proc的文件内容。
```
-[namespace-process]-# ls /proc
1     1537  198   212   2487  2818  29    3131  329   3363  3422  3557  3793  4250  492   5273  595  62    6519  72    80    87   919        crypto       kmsg          schedstat          vmstat
10    16    1983  213   2501  2841  2902  3139  3297  3367  3425  3571  38    430   493   53    598  6220  654   7221  8042  88   922        devices      kpagecgroup   scsi               zoneinfo
1025  17    2     2133  2506  2846  2913  3156 ....
```
这里输出的结果很多，因为**/proc**还是宿主机的，下面将**/proc** mount到我们自己的Namespace下面来：
```
-[namespace-process]-# mount -t proc proc /proc
-[namespace-process]-# ls /proc
1       buddyinfo  consoles  diskstats    fb           iomem     kcore      kpagecgroup  locks    modules  pagetypeinfo  schedstat  softirqs  sysrq-trigger  tty                vmallocinfo
4       bus        cpuinfo   dma          filesystems  ioports   key-users  kpagecount   mdstat   mounts   partitions    scsi       stat      sysvipc        uptime             vmstat
acpi    cgroups    crypto    driver       fs           irq       keys       kpageflags   meminfo  mtrr     pressure      self       swaps     thread-self    version            zoneinfo
asound  cmdline    devices   execdomains  interrupts   kallsyms  kmsg       loadavg      misc     net      sched_debug   slabinfo   sys       timer_list     version_signature
```
结果一下子少了很多，这里我们就可以用**ps**来查看系统的进程了。
```
-[namespace-process]-# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 17:07 pts/1    00:00:00 sh
root         5     1  0 17:11 pts/1    00:00:00 ps -ef
```
可以看到，当前的Namespace中，sh进程是PID为1的进程。  

## 增加Network Namespace
**Network Namespace**是用来隔离网络设备、IP地址端口等网络栈的Namespace。Network Namespace可以让每个容器拥有自己独立（虚拟的）网络设备，而且容器内的应用可以绑定到自己的端口，每个Namespace内的端口都不会互相冲突。在宿主机上搭建网桥后，就能很方便地实现容器之间通讯，而且不同容器上的应用可以使用相同的端口。  
继续上述代码，加入`CLONE_NEWNET`：
```Golang
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
		Cloneflags:
			syscall.CLONE_NEWUTS |
			syscall.CLONE_NEWUSER |
			syscall.CLONE_NEWIPC |
			syscall.CLONE_NEWPID|
			syscall.CLONE_NEWNS |
			syscall.CLONE_NEWNET, // 增加Network Namespace
		UidMappings: []syscall.SysProcIDMap{
			{
				ContainerID: 0,
				HostID:      os.Getuid(),
				Size:        1,
			},
		},
		GidMappings: []syscall.SysProcIDMap{
			{
				ContainerID: 0,
				HostID:      os.Getgid(),
				Size:        1,
			},
		},
	}

	// set identify for this demo
	cmd.Env = []string{"PS1=-[namespace-process]-# "}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```
运行我们的程序，查看网络设备，发现为空
```
-[namespace-process]-# ifconfig
-[namespace-process]-#
```
在宿主机上查看网络设备，发现有lo, enp7s0这些网络设备。
```
enp7s0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether 98:fa:9b:f0:85:c2  txqueuelen 1000  (以太网)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (本地环回)
        RX packets 16381  bytes 23729834 (23.7 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 16381  bytes 23729834 (23.7 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
....
```
从上面的结果我们可以看出Network是隔离了。