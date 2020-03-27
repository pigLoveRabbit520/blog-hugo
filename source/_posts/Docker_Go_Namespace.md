title: 一步步自己做个Docker之Go调用Namespace
author: Salamander
tags:
  - Docker
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