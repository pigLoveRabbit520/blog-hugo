---
title: Linux网络虚拟化技术之Veth和Bridge
author: Salamander
tags:
  - linux
  - network
categories:
  - linux
date: 2020-01-14 13:00:00
---
## Veth
Veth缩写是Virtual ETHernet。veth设备是在linux内核中是成对出现（所以也叫`veth-pair`），两个设备彼此相连，一个设备从协议栈读取数据后，会将数据发送到另一个设备上去。这个设备其实是专门为`container`所建的，作用就是把一个**network namespace**发出的数据包转发到另一个**namespace**（通常就是宿主机）。    
![](https://s2.ax1x.com/2020/01/14/lbBga9.png)  

<!-- more -->


### 添加Veth设备
```
$ sudo ip netns add net0
$ sudo ip netns add net1
$ sudo ip link add veth0 netns net0 type veth peer name veth1 netns net1 #添加 veth 设备对
```
上面的命令将创建两个命名空间net0和net1，以及一对veth设备，并将veth1分配给命名空间net0，将veth2分配给命名空间net1。这两个名称空间与此VETH对相连。分配一对IP地址，你就可以ping通两者之间的通讯。

```
$ ip netns ls  # 查看创建的network namespace
net1
net0
$ sudo ip netns exec net0 ip addr # 查看net0下的网络设备
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: veth0@if2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether d6:6e:4f:fb:6b:76 brd ff:ff:ff:ff:ff:ff link-netnsid 0

$ sudo ip netns exec net1 ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: veth1@if2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 92:05:82:e6:da:73 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```
我们给这对 veth pair 配置上 ip 地址，并启用它们以及 lo 接口:
```
sudo ip netns exec net0 ip link set veth0 up
sudo ip netns exec net0 ip addr add 10.0.1.1/24 dev veth0
sudo ip netns exec net0 ip link set lo up
sudo ip netns exec net0 ip route
10.0.1.0/24 dev veth0 proto kernel scope link src 10.0.1.1 linkdown 


sudo ip netns exec net1 ip link set veth1 up
sudo ip netns exec net1 ip addr add 10.0.1.2/24 dev veth1
sudo ip netns exec net1 ip link set lo up
sudo ip netns exec net1 ip route
10.0.1.0/24 dev veth1 proto kernel scope link src 10.0.1.2
```
可以看到，在每个 namespace 中，在配置了 ip 之后，还自动生成了对应的
路由表信息，网络 10.0.1.0/24 数据报文都会通过 vethpair 进行传输。下面使用 ping 命令 可以验证它们的连通性，并在 veth0 和 veth1 上抓包：
```
sudo ip netns exec net0 ping -c 3 10.0.1.2
```
![](https://s2.ax1x.com/2020/01/14/lb4nOA.png)


## Bridge
Bridge（桥）是 Linux 上用来做 TCP/IP 二层协议交换的设备，与现实世界中的交换机功能相似。Bridge 设备实例可以和 Linux 上其他网络设备实例连接，既 attach 一个从设备，类似于在现实世界中的交换机和一个用户终端之间连接一根网线。当有数据到达时，Bridge 会根据报文中的 MAC 信息进行广播、转发、丢弃处理。

![](https://s2.ax1x.com/2020/01/14/lb48fS.png)

使用Bridge前，需要安装`bridge-utils`包
```
sudo apt install bridge-utils
```
### 查看bridge
```
$ brctl show

bridge name	bridge id		STP enabled	interfaces
br-1f7059361887		8000.0242740c4703	no		veth35716a0
							vethc51a0aa
							vethd17adab
br-4646ac4e576a		8000.02421afdd01b	no		
br-4b05476c9f71		8000.024240053946	no		
br-5282ac3290df		8000.0242ad5bae1e	no		
br-638972aaac40		8000.024278b6d203	no		
br-67973b91b458		8000.024279f6c039	no		
br-96dbd98373e7		8000.0242f23b4758	no		veth13b0e48
							veth3af358a
							veth52b951a
							veth5adc514
							veth6e141f8
							veth6fcb89b
							veth710d968
							vethc9477cd
							vethcf35aff
docker0		8000.0242fee4c327	no		
```
上面这个是`docker0`的是Docker给你创建的bridge（如果你设备上装有docker的话 就可以看到）。  

### 创建一个bridge
```
brctl addbr br66
```

上面命令创建一个名为br66的桥。  
接下来我们可以把已有的网络设备绑定到这个桥上，在这之前可以看看我们有有哪些网卡接口，可以用
```
ip addr show
```
假设上面查出来，有eth0和eth1两个网卡接口，下面我们把他们用命令绑定到一起
```
brctl addif br0 eth0 eth1 # eth0和eth1的顺序不重要，不影响结果
```
再来查看绑定关系
```
$ brctl show
bridge name     bridge id               STP enabled     interfaces
br66             8000.001ec952d26b       yes             eth0
                                                        eth1
```
就是说eth0和eth1绑定到了br0这个桥上了。













## 参考文章
[Linux 上的基础网络设备详解](https://www.ibm.com/developerworks/cn/linux/1310_xiawc_networkdevice/index.html)