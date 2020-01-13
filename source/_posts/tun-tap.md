title: Linux网络虚拟化技术之tun/tap
author: Salamander
tags:
  - network
  - linux
  - tun
  - tap
categories:
  - linux
date: 2020-01-13 15:00:00
---
我们都知道，Linux实际是通过**网络设备**去操作和使用网卡的，系统安装了一个网卡之后会为其生成一个网络设备实例，比如**eth0**（或者叫**enp7s0**，不同发行版默认网卡命名规则不同）。随着网络虚拟化技术的发展，Linux支持创建出虚拟化的设备，可以通过虚拟化设备的组合实现多种多样的功能和网络拓扑。  
常见的虚拟化设备有**tun/tap**、**Veth**、**Bridge**、**802.1q VLAN device**。  

本文环境：
* OS：Ubuntu 18.04.3 LTS

先回顾一下经典的**OSI**七层网络模型：  
┌───────┐  
  │　应用层　│←第七层  
├───────┤  
│　表示层　│  
├───────┤  
│　会话层　│  
├───────┤  
│　传输层　│  
├───────┤  
│　网络层　│   
├───────┤  
│数据链路层│  
├───────┤  
│　物理层　│←第一层  
└───────┘ 

OSI七层参考模型

## 虚拟设备和物理设备的区别
对于一个网络设备来说，就像一个管道（pipe）一样，**有两端**，从其中任意一端收到的数据将从另一端发送出去。  

比如一个物理网卡eth0，它的两端分别是内核协议栈（通过内核网络设备管理模块间接的通信）和外面的物理网络，从物理网络收到的数据，会转发给内核协议栈，而应用程序从协议栈发过来的数据将会通过物理网络发送出去。  

那么对于一个虚拟网络设备呢？首先它也归内核的网络设备管理子系统管理，对于Linux内核网络设备管理模块来说，虚拟设备和物理设备没有区别，都是网络设备，都能配置IP，从网络设备来的数据，都会转发给协议栈，协议栈过来的数据，也会交由网络设备发送出去，至于是怎么发送出去的，发到哪里去，那是设备驱动的事情，跟Linux内核就没关系了，所以说虚拟网络设备的一端也是协议栈，而另一端是什么取决于虚拟网络设备的驱动实现。

## tun/tap
### tun分析实验
先上图说话：  
![](https://s2.ax1x.com/2020/01/13/l7D6zT.png)

上图中是**tun**设备的数据走向。
图中**nsfocus_tun0**就是tun0，是一个tun/tap虚拟设备，而**eno16777736**就是eth0。  
socket、协议栈（Newwork Protocol Stack）和网络设备（eth0和tun0）部分都在内核层，其实socket是协议栈的一部分，这里分开来的目的是为了看的更直观。

从上图中可以看出它和物理设备eth0的差别，它们的一端虽然都连着协议栈，但另一端不一样，eth0的另一端是物理网络，这个物理网络可能就是一个交换机，而tun0的另一端是一个用户层的程序，协议栈发给tun0的数据包能被这个应用程序读取到，并且应用程序能直接向tun0写数据。  

数据流向分析：
1. User Application A通过套接字（socket A）发数据发给使用与**eno16777736**处于同一个网段ip的应用程序，数据走向为通过socket A发给协议栈，最后通过netdevice子系统中的eno16777736的设备驱动（以太网驱动）发送出去，这个是通过真实的物理网卡发送出去。
2. User Application B通过套接字（socket B）发送数据给使用与**nsfocus_tun0**处于同一个网段ip的应用程序，数据走向为通过socket B发送给协议栈，最后通过netdevice子系统中的**nsfocus_tun0**的设备驱动（tun驱动）发送出去。由于tun设备没有对应真实的物理网卡，所以nsfocus_tun0对端收取数据的是User Application C。User Application C通过读写/dev/tun设备文件进行数据的收发。

其实一般**User Application C**就是个VPN程序（例如openvpn），它收到数据包之后，做一些跟业务相关的处理，然后构造一个新的数据包，将原来的数据包嵌入在新的数据包中，最后通过socket B将数据包转发出去，这时候新数据包的源地址变成了eth0的地址，而目的IP地址变成了一个其它的地址，比如是10.33.0.1（VPN服务器地址），协议栈根据本地路由，发现这个数据包应该要通过**eth0**发送出去，于是将数据包交给eth0，最后**eth0**通过物理网络将数据包发送出去。


从上面的流程中可以看出，数据包选择走哪个网络设备完全由**路由表**控制，所以如果我们想让某些网络流量走应用程序B的转发流程，就需要配置路由表让这部分数据走tun0。



## 示例程序
为了使用tun/tap设备，用户层程序需要通过系统调用打开/dev/net/tun获得一个读写该设备的文件描述符(FD)，并且调用ioctl()向内核注册一个TUN或TAP类型的虚拟网卡(实例化一个tun/tap设备)，其名称可能是**tap7b7ee9a9-c1/vnetXX/tunXX/tap0**等。

这里写了一个程序，它收到tun设备的数据包之后，只打印出收到了多少字节的数据包，其它的什么都不做。
```
#include <net/if.h>
#include <sys/ioctl.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>
#include <sys/types.h>
#include <linux/if_tun.h>
#include<stdlib.h>
#include<stdio.h>

int tun_alloc(int flags)
{

    struct ifreq ifr;
    int fd, err;
    char *clonedev = "/dev/net/tun";

    if ((fd = open(clonedev, O_RDWR)) < 0) {
        return fd;
    }

    memset(&ifr, 0, sizeof(ifr));
    ifr.ifr_flags = flags;

    if ((err = ioctl(fd, TUNSETIFF, (void *) &ifr)) < 0) {
        close(fd);
        return err;
    }

    printf("Open tun/tap device: %s for reading...\n", ifr.ifr_name);

    return fd;
}

int main()
{

    int tun_fd, nread;
    char buffer[1500];

    /* Flags: IFF_TUN   - TUN device (no Ethernet headers)
     *        IFF_TAP   - TAP device
     *        IFF_NO_PI - Do not provide packet information
     */
    tun_fd = tun_alloc(IFF_TUN | IFF_NO_PI);

    if (tun_fd < 0) {
        perror("Allocating interface");
        exit(1);
    }

    while (1) {
        nread = read(tun_fd, buffer, sizeof(buffer));
        if (nread < 0) {
            perror("Reading from interface");
            close(tun_fd);
            exit(1);
        }

        printf("Read %d bytes from tun/tap device\n", nread);
    }
    return 0;
}
```
编译、运行程序，会发现多出一个网络设备
```
$ gcc -o tun tun.c
$ sudo ./tun
Open tun/tap device: tun0 for reading...


$ ip addr
...
8: tun0: <POINTOPOINT,MULTICAST,NOARP> mtu 1500 qdisc noop state DOWN group default qlen 500
    link/none
```
`tun0`就是新增的网络设备，现在给它配置一个ip，查看接口信息
```
$ sudo ifconfig tun0 192.168.10.11/24
$ ip addr
...
8: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 192.168.10.11/24 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fe80::dd59:736:65ee:e31a/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
```
这时候我们ping地址`192.168.10.12`
```
ping 192.168.10.12 -c 2
```
发现`tun`程序收到了数据
![](https://s2.ax1x.com/2020/01/13/l7oJfS.png)


### tun和tap区别
两者很类似，只是tun和tap设备他们工作的协议栈层次不同，tap等同于一个以太网设备，用户层程序向tap设备读写的是二层数据包如以太网数据帧，tap设备最常用的就是作为虚拟机网卡。tun则模拟了网络层设备，操作第三层数据包比如IP数据包，openvpn使用TUN设备在C/S间建立VPN隧道。


### tap分析实验
tap设备最常见的用途就是作为虚拟机网卡。











## 参考文章
* [Linux虚拟网络设备之tun/tap](https://segmentfault.com/a/1190000009249039)
* [云计算底层技术-虚拟网络设备(tun/tap,veth)
](https://opengers.github.io/openstack/openstack-base-virtual-network-devices-tuntap-veth/)
