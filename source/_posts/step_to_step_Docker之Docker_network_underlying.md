title: 一步步自己做个Docker之Docker网络原理
author: Salamander
tags:
  - Docker
  - Linux
categories:
  - Docker
  - Linux
date: 2020-04-28 15:00:00
---
![docker logo](/images/docker-logo.png)

本文环境：
* OS：Ubuntu 18.04.4 LTS
* Golang版本：1.12.13


<!-- more -->


## 自己创建Docker网络

>当 Docker 启动时，会自动在主机上创建一个 docker0 虚拟网桥，实际上是 Linux 的一个 bridge，可以理解为一个软件交换机。它会在挂载到它的网口之间进行转发。 同时，Docker 随机分配一个本地未占用的私有网段（在 RFC1918 中定义）中的一个地址给 docker0 接口。比如典型的 172.17.42.1，掩码为 255.255.0.0。此后启动的容器内的网口也会自动分配一个同一网段（172.17.0.0/16）的地址。 当创建一个 Docker 容器的时候，同时会创建了一对 veth pair 接口（当数据包发送到一个接口时，另外一个接口也可以收到相同的数据包）。这对接口一端在容器内，即 eth0；另一端在本地并被挂载到 docker0 网桥，名称以 veth 开头（例如 vethAQI2QT）。通过这种方式，主机可以跟容器通信，容器之间也可以相互通信。Docker 就创建了在主机和所有容器之间一个虚拟共享网络。如图
![upload successful](https://s1.ax1x.com/2020/04/29/J7aODe.png)


下面以自定义的容器方式，一步步配置网络, 达到以下目标:
* 容器间能够通信
* 容器能够联外网
首先创建一个容器，但不使用默认网络配置，使用`--net=none`选项:
```
docker run -t -i --net=none ubuntu:14.04 bash
docker ps # 获取容器id=6414d7278905
```
获取容器pid:
```
docker inspect 6414d7278905 | grep -i "\<pid\""
#  "Pid": 11776,
pid=11776
```
创建一个新的netns，并把容器放入新建的netns中。（根据约定，命名的 network namespace 是可以打开的 **/var/run/netns/** 目录下的一个对象。比如有一个名称为 net1 的 network namespace 对象，则可以由打开 **/var/run/netns/net1** 对象产生的文件描述符引用 network namespace net1。通过引用该文件描述符，可以修改进程的 network namespace。）
```
sudo ip netns add netns666 # 会产生一个/var/run/netns/netns666的文件
sudo ip netns ls # 查看新创建netns
netns666
sudo mount --bind /proc/$pid/ns/net /var/run/netns/netns666 # 加入新的netns
sudo ip netns pids netns666   # 查看netns666中进程的 PID
11776
```

接下来创建一个veth对，其中一个设置为容器所在的netns，即`netns666`
```
sudo ip link add name veth_d366 type veth peer name veth_d366_peer
sudo ip link set veth_d366_peer netns netns666
```
进入`netns666` netns设置网卡名称和ip:
```
sudo ip netns exec netns666 bash
sudo ip link set veth_d366_peer name eth0
sudo ifconfig  eth0 10.0.0.2/24 # 设置ip为10.0.0.2
ping 10.0.0.2 # 能ping通
exit
```
上述命令给veth_d366_peer配置完ip的情况：
```
ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
36: eth0@if37: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000
    link/ether 9e:67:30:e3:65:34 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.2/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
```
在容器中ping 10.0.0.2也能ping通,说明设置正确
```
ping 10.0.0.2 # 宿主机上应该不通
docker exec 6414d7278905 ping 10.0.0.2 # 成功ping通
```
创建网桥，并把veth另一端的虚拟网卡加入新创建的网桥中:
```
sudo brctl addbr br666 # 创建新网桥br666
sudo brctl addif br666 veth_d366 # 把虚拟网卡加入网桥br666中
sudo ifconfig br666 10.0.0.1/24 # 设置网桥ip
sudo ip link set veth_d366 up # 启动虚拟网卡
```
测试下：
```
ping 10.0.0.2 # 宿主机上成功ping通
docker exec 6414d7278905 ping 10.0.0.1 # 成功ping通
```
若以上两个都能ping通说明配置成功！  
最后，我们需要使得容器能够联外网，需要设置NAT，使用iptables设置:  
```
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o em1 -j MASQUERADE
```
`em1`是真正的物理网卡对应的网络设备，不同电脑上名字不一样，我的Ubuntu上为`enp7s0`。  
另外，还需要设置`FORWARD`规则（允许`br666`转发，IP forwarding要开启，使用`sudo sysctl -w net.ipv4.ip_forward=1`）
```
sudo iptables -A FORWARD -i br666 -o br666  -j ACCEPT
sudo iptables -A FORWARD -i br666 ! -o br666  -j ACCEPT
sudo iptables -A FORWARD ! -i br666  -o br666  -j ACCEPT
```


设置容器默认路由为网桥ip（注意在容器内使用route add 添加, 会出现SIOCADDRT: Operation not permitted错误), 因此只能使用ip netns exec设置:
```
sudo ip netns exec netns666 route add default gw 10.0.0.1
```
测试，此时请确保宿主机能够联外网,进入容器内部:
```
ping baidu.com # 成功ping通，确保icmp没有被禁
```
效果图：  

![upload successful](/images/my_docker_ping.png)


## Docker网络和虚拟机网络区别

### 虚拟机
虚拟机，如图，虚拟机通过tun/tap或者其它类似的虚拟网络设备，将虚拟机内的网卡同br0连接起来，这样就达到和真实交换机一样的效果，虚拟机发出去的数据包先到达br0，然后由br0交给eth0发送出去，数据包都不需要经过host机器的协议栈，效率高。  

![image](https://s1.ax1x.com/2020/04/30/Jb090O.png)

### Docker
docker，如图，由于容器运行在自己单独的network namespace里面，所以都有自己单独的协议栈，情况和上面的虚拟机差不多，但它采用了另一种方式来和外界通信

![](https://s1.ax1x.com/2020/04/30/JbrFNF.png)

容器中配置网关为.9.1，发出去的数据包先到达`br0`，然后交给host机器的协议栈，由于目的IP是外网IP，且host机器开启了`IP forward`功能，于是数据包会通过eth0发送出去，由于.9.1是内网IP，所以一般发出去之前会先做NAT转换（NAT转换和IP forward功能都需要自己配置）。**由于要经过host机器的协议栈，并且还要做NAT转换，所以性能没有上面虚拟机那种方案好**，优点是容器处于内网中，安全性相对要高点。（由于数据包统一由IP层从eth0转发出去，所以不存在mac地址的问题，在无线网络环境下也工作良好）




参考：
* [Linux ip netns 命令](https://www.cnblogs.com/sparkdev/p/9253409.html)
* [docker网络原理.md](https://github.com/int32bit/notes/blob/master/cloud/docker%E7%BD%91%E7%BB%9C%E5%8E%9F%E7%90%86.md)
* [Docker 网络之理解 bridge 驱动](https://www.cnblogs.com/sparkdev/p/9217310.html)
* [Linux内核网络设备——bridge设备](http://blog.nsfocus.net/linux-bridge/)