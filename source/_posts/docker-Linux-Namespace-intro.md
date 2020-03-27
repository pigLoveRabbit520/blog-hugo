title: 一步步自己做个Docker之Linux Namespace 简介
author: Salamander
tags:
  - Docker
  - Namespace
  - Cgroup
categories:
  - Docker
date: 2019-11-28 16:10:00
---
![docker logo](/images/docker-logo.png)


本文环境：
* OS：Ubuntu 18.04.3 LTS
* 内核版本： 5.0.0-36-generic 

## Linux Namespaces
Docker的所用的两个关键技术，一个是`Namespaces`，一个是`Cgroups`。它俩都不是新技术，Linux内核很早就支持，但是Docker把它们有机地结合起来，加上自己创新，使得现在容器技术非常流行。  
`Linux Namespaces`其实是做到了进程之间全局资源的隔离，譬如，`UTS Namespace`隔离了Hostname空间。这意味着在新的`UTS Namespace`中的进程，可以拥有不同于宿主机的主机名。 

<!-- more -->

目前Linux内核主要实现了以下几种不同的资源`Namespace`：

| 名称 | 宏定义 | 隔离的内容 |
|--- |--- |--- |
|IPC|CLONE_NEWIPC|System V IPC, POSIX message queues (since Linux 2.6.19)|
|Network|CLONE_NEWNET|network device interfaces, IPv4 and IPv6 protocol stacks, IP routing tables, firewall rules, the /proc/net and /sys/class/net directory trees, sockets, etc (since Linux 2.6.24)|
|Mount|CLONE_NEWNS|Mount points (since Linux 2.4.19)|
|PID|CLONE_NEWPID|Process IDs (since Linux 2.6.24)|
|User|CLONE_NEWUSER|User and group IDs (started in Linux 2.6.23 and completed in Linux 3.8)|
|UTS|CLONE_NEWUTS|Hostname and NIS domain name (since Linux 2.6.19)|
|Cgroup|CLONE_NEWCGROUP|Cgroup root directory (since Linux 4.6)|

要注意一点的是，不是所有的系统资源都能隔离，时间就是个例外，没有对应的`Namespace`，因此同一台Linux启动的容器时间都是相同的。

### 尝试一下Namespace
```
lucy@lucy-computer:~$ unshare -h

用法：
 unshare [选项] [<程序> [<参数>...]]

以某些未与父(进程)共享的名字空间运行某个程序。

选项：
 -m, --mount[=<文件>]      取消共享 mounts 名字空间
 -u, --uts[=<文件>]        取消共享 UTS 名字空间(主机名等)
 -i, --ipc[=<文件>]        取消共享 System V IPC 名字空间
 -n, --net[=<file>]        取消共享网络名字空间
 -p, --pid[=<文件>]        取消共享 pid 名字空间
 -U, --user[=<文件>]       取消共享用户名字空间
 -C, --cgroup[=<文件>]     取消共享 cgroup 名字空间
 -f, --fork                在启动<程序>前 fork
     --mount-proc[=<目录>] 先挂载 proc 文件系统(连带打开 --mount)
 -r, --map-root-user       将当前用户映射为 root (连带打开 --user)
     --propagation slave|shared|private|unchanged
                           修改 mount 名字空间中的 mount 传播
 -s, --setgroups allow|deny  控制用户名字空间中的 setgroups 系统调用

 -h, --help                display this help
 -V, --version             display version
```
`unshare`命令可以让你在新的名称空间集中启动一个新的程序（unshared本身的含义就是不和父进程共享）。  
下面的例子使用了`UTS namespace`，可以看到在新的`/bin/sh`进程中修改hostname，并没有影响宿主机：
```
$ sudo su                   # become root user
$ hostname                  # check current hostname
lucy-computer  
$ unshare -u /bin/sh        # create a shell in new UTS namespace
$ hostname my-new-hostname  # set hostname
$ hostname                  # confirm new hostname
my-new-hostname  
$ exit                      # exit new UTS namespace
$ hostname                  # confirm original hostname unchanged
lucy-computer
```

### 三个系统调用
`unshare`命令很棒，但是当我们想要对程序中的命名空间进行更细粒度的控制时，那该怎么办呢？  
Linux 内核提供的功能都会提供`系统调用`接口供应用程序使用，`Namespace`也不例外。和`Namespace`相关的系统调用主要有三个：
* [clone](http://man7.org/linux/man-pages/man2/clone.2.html)
* [setns](http://man7.org/linux/man-pages/man2/setns.2.html)
* [unshare](http://man7.org/linux/man-pages/man2/unshare.2.html)

**注意**：这些系统调用都是 linux 内核实现的，不能直接适用于其他操作系统。

查看一下它们对应的C语言函数原型：
#### clone：创建新进程并设置它的Namespace
`clone`类似于`fork`系统调用，可以创建一个新的进程，不同的是你可以指定要子进程要执行的函数以及通过参数控制子进程的运行环境。

> 实际上，clone() 是在 C 语言库中定义的一个封装(wrapper)函数，它负责建立新进程的堆栈并且调用对编程者隐藏的 clone() 系统调用。Clone() 其实是 linux 系统调用 fork() 的一种更通用的实现方式，它可以通过 flags 来控制使用多少功能。

```
#define _GNU_SOURCE
#include <sched.h>

int clone(int (*fn)(void *), void *child_stack, int flags, void *arg);
```
* fn：指定一个由新进程执行的函数。当这个函数返回时，子进程终止。该函数返回一个整数，表示子进程的退出代码。
* child_stack：传入子进程使用的栈空间，也就是把用户态堆栈指针赋给子进程的 esp 寄存器。调用进程(指调用 clone() 的进程)应该总是为子进程分配新的堆栈。
* flags：表示使用哪些 CLONE_ 开头的标志位，与 namespace 相关的有CLONE_NEWIPC、CLONE_NEWNET、CLONE_NEWNS、CLONE_NEWPID、CLONE_NEWUSER、CLONE_NEWUTS 和 CLONE_NEWCGROUP，如果要同时隔离多个 namespace，**可以使用 | (按位或)组合这些参数**。
* arg：指向传递给 fn() 函数的参数。

#### setns：让进程加入已经存在Namespace
setns 能够把某个进程加入到给定的 namespace，它的定义是这样的：
```
#define _GNU_SOURCE
#include <sched.h>
int setns(int fd, int nstype);
```
和`clone()`函数一样，C 语言库中的`setns()`函数也是对`setns系统调用`的封装。  
* fd：表示要加入 namespace 的文件描述符。它是一个指向 /proc/[pid]/ns 目录中文件的文件描述符，可以通过直接打开该目录下的链接文件或者打开一个挂载了该目录下链接文件的文件得到。
* nstype：参数 nstype 让调用者可以检查 fd 指向的 namespace 类型是否符合实际要求。若把该参数设置为 0 表示不检查。

#### unshare：让进程加入新的Namespace
```
#define _GNU_SOURCE
#include <sched.h>
int unshare(int flags);
```
`unshare()`函数比较简单，只有一个参数`flags`，它的含义和`clone()`的`flags`相同。`unshare`和 `setns` 的区别是，`setns` 只能让进程加入到已经存在的`namespace`中，而`unshare`则让进程离开当前的`namespace`，加入到新建的`namespace`中。  

`unshare()`和`clone()`的区别在于：`unshare()`是把当前进程进入到新的`namespace`；`clone()`是创建新的进程，然后让新创建的进程（子进程）加入到新的`namespace`。


## C程序中使用clone系统调用
我们先来看看 clone 一个简单的使用例子：创建一个新的进程，并执行 /bin/bash，这样就可以接受命令，方便我们查看新进程的信息。
```
#define _GNU_SOURCE
#include <sched.h>
#include <sys/wait.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

// 设置子进程要使用的栈空间
#define STACK_SIZE (1024*1024)
static char container_stack[STACK_SIZE];

#define errExit(code, msg); {if(code == -1){perror(msg); exit(-1);} }


char* const container_args[] = {
    "/bin/bash",
    NULL
};

static int container_func(void *arg)
{
    pid_t pid = getpid();
    printf("Container[%d] - inside the container!\n", pid);

    // 用一个新的bash来替换掉当前子进程，
    // 这样我们就能通过 bash 查看当前子进程的情况.
    // bash退出后，子进程执行完毕
    execv(container_args[0], container_args);

    // 从这里开始的代码将不会被执行到，因为当前子进程已经被上面的bash替换掉了;
    // 所以如果执行到这里，一定是出错了
    printf("Container[%d] - oops!\n", pid);
    return 1;
}


int main(int argc, char *argv[])
{
    pid_t pid = getpid();
    printf("Parent[%d] - create a container!\n", pid);

    // 创建并启动子进程，调用该函数后，父进程将继续往后执行，也就是执行后面的waitpid
    pid_t child_pid = clone(container_func,  // 子进程将执行container_func这个函数
                    container_stack + sizeof(container_stack),
                    // 这里SIGCHLD是子进程退出后返回给父进程的信号，跟namespace无关
                    SIGCHLD,
                    NULL);  // 传给child_func的参数
    errExit(child_pid, "clone");

    waitpid(child_pid, NULL, 0); // 等待子进程结束

    printf("Parent[%d] - container exited!\n", pid);
    return 0;
}
```
这段代码不长，但是做了很多事情：
* 通过`clone()`创建出一个子进程，并设置启动时的参数
* 在子进程中调用 execv 来执行 /bin/bash，等待用户进行交互
* 子进程退出之后，父进程也跟着退出

我们可以用`ls -l /proc/$$/ns`查看当前进程所在命名空间的信息，运行程序：
```
lucy@lucy-computer:~$ gcc container.c -o container
lucy@lucy-computer:~$ ./container 
Parent[19644] - create a container!
Container[19645] - inside the container!
lucy@lucy-computer:~$ ls -l /proc/$$/ns
总用量 0
lrwxrwxrwx 1 lucy lucy 0 11月 28 15:36 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 lucy lucy 0 11月 28 15:36 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 lucy lucy 0 11月 28 15:36 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 lucy lucy 0 11月 28 15:36 net -> 'net:[4026531992]'
lrwxrwxrwx 1 lucy lucy 0 11月 28 15:36 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 lucy lucy 0 11月 28 15:36 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 lucy lucy 0 11月 28 15:36 user -> 'user:[4026531837]'
lrwxrwxrwx 1 lucy lucy 0 11月 28 15:36 uts -> 'uts:[4026531838]'
lucy@lucy-computer:~$ exit
exit
Parent[19644] - container exited!
lucy@lucy-computer:~$ ls -l /proc/$$/ns
总用量 0
lrwxrwxrwx 1 lucy lucy 0 11月 28 15:39 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 lucy lucy 0 11月 28 15:39 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 lucy lucy 0 11月 28 15:39 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 lucy lucy 0 11月 28 15:39 net -> 'net:[4026531992]'
lrwxrwxrwx 1 lucy lucy 0 11月 28 15:39 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 lucy lucy 0 11月 28 15:39 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 lucy lucy 0 11月 28 15:39 user -> 'user:[4026531837]'
lrwxrwxrwx 1 lucy lucy 0 11月 28 15:39 uts -> 'uts:[4026531838]'
```
各类命名空间id都是一样，因为我们只是单单使用了`clone`，未设置要隔离的命名空间，现在，我们加入`UTS Namespace`隔离，`UTS namespace` 功能最简单，它只隔离了 hostname 和 NIS domain name 两个资源。  
同一个 namespace 里面的进程看到的 hostname 和 domain name 是相同的，这两个值可以通过 `sethostname(2)` 和 `setdomainname(2)` 来进行设置，也可以通过 `uname(2)`、`gethostname(2)` 和 `getdomainname(2)` 来读取。    
**注意**： UTS 的名字来自于`uname`函数用到的结构体`struct utsname`，这个结构体的名字源自于`UNIX Time-sharing System`。  
代码主要修改两个地方：clone 的参数加上了 CLONE_NEWUTS，子进程函数中使用`sethostname`来设置 hostname。  

```
#define _GNU_SOURCE
#include <sched.h>
#include <sys/wait.h>
#include <sys/utsname.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

// 设置子进程要使用的栈空间
#define STACK_SIZE (1024*1024)
static char container_stack[STACK_SIZE];

#define errExit(code, msg); {if(code == -1){perror(msg); exit(-1);} }


char* const container_args[] = {
    "/bin/bash",
    NULL
};

static int container_func(void *hostname)
{
    pid_t pid = getpid();
    printf("Container[%d] - inside the container!\n", pid);

    // 使用 sethostname 设置子进程的 hostname 信息
    struct utsname uts;
    if (sethostname(hostname, strlen(hostname)) == -1) {
        errExit(-1, "sethostname")
    };

    // 使用 uname 获取子进程的机器信息，并打印 hostname 出来
    if (uname(&uts) == -1){
        errExit(-1, "uname")
    }
    printf("Container[%d] - container uts.nodename: [%s]!\n", pid, uts.nodename);

    // 用一个新的bash来替换掉当前子进程，
    // 这样我们就能通过 bash 查看当前子进程的情况.
    // bash退出后，子进程执行完毕
    execv(container_args[0], container_args);

    // 从这里开始的代码将不会被执行到，因为当前子进程已经被上面的bash替换掉了;
    // 所以如果执行到这里，一定是出错了
    printf("Container[%d] - oops!\n", pid);
    return 1;
}


int main(int argc, char *argv[])
{
    pid_t pid = getpid();
    printf("Parent[%d] - create a container!\n", pid);

    // 把第一个参数作为子进程的 hostname，默认是 `container`
    char *hostname;
    if (argc < 2) {
        hostname = "container";
    } else {
        hostname = argv[1];
    }

    // 创建并启动子进程，调用该函数后，父进程将继续往后执行，也就是执行后面的waitpid
    pid_t child_pid = clone(container_func,  // 子进程将执行container_func这个函数
                    container_stack + sizeof(container_stack),
                    // CLONE_NEWUTS表示创建新的UTS namespace
                    CLONE_NEWUTS | SIGCHLD,
                    hostname);  // 传给child_func的参数
    errExit(child_pid, "clone");

    waitpid(child_pid, NULL, 0); // 等待子进程结束

    printf("Parent[%d] - container exited!\n", pid);
    return 0;
}
```
执行程序，发现容器中hostname与宿主机已经不一样了，容器中`UTS Namespace`id也跟宿主机不一样了（这里需要root权限）：

```
sudo su
root@lucy-computer:/home/lucy# gcc container.c -o container
root@lucy-computer:/home/lucy# ./container 
Parent[21091] - create a container!
Container[21092] - inside the container!
Container[21092] - container uts.nodename: [container]!
root@container:/home/lucy# hostname
container
root@container:/home/lucy# ls -l /proc/$$/ns
总用量 0
lrwxrwxrwx 1 root root 0 11月 28 16:00 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 11月 28 16:00 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 11月 28 16:00 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0 11月 28 16:00 net -> 'net:[4026531992]'
lrwxrwxrwx 1 root root 0 11月 28 16:00 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 11月 28 16:00 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 11月 28 16:00 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 11月 28 16:00 uts -> 'uts:[4026532944]'
root@container:/home/lucy# exit
exit
Parent[21091] - container exited!
root@lucy-computer:/home/lucy# ls -l /proc/$$/ns
总用量 0
lrwxrwxrwx 1 root root 0 11月 28 16:00 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 11月 28 16:00 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 11月 28 16:00 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0 11月 28 16:00 net -> 'net:[4026531992]'
lrwxrwxrwx 1 root root 0 11月 28 16:00 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 11月 28 16:00 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 11月 28 16:00 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 11月 28 16:00 uts -> 'uts:[4026531838]'
root@lucy-computer:/home/lucy# hostname
lucy-computer
```



### Let's Go
C语言很底层，能控制到很多细节，但是它对于大部分人有点困难，接下来我们会有Go语言来一步步实现Docker容器。







### 参考资料
* [cizixs.com/2017/08/29/linux-namespace](https://cizixs.com/2017/08/29/linux-namespace/)
* [Linux Namespace : 简介](https://www.cnblogs.com/sparkdev/p/9365405.html)