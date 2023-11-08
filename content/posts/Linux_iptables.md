---
title: Linux之iptables
author: Salamander
tags:
  - Linux
  - iptables
categories:
  - Linux
date: 2020-04-26 13:00:00
---
## 简介
管理网络流量是系统管理员必需处理的最棘手工作之一，我们必需规定连接系统的用户满足防火墙的传入和传出要求，以最大限度保证系统免受攻击。`iptables`正是这样的工具。

其实不是真正的防火墙，我们可以把它理解成一个客户端代理，用户通过iptables这个代理，将用户的安全设定执行到对应的"安全框架"中，这个"安全框架"才是真正的防火墙，这个框架的名字叫**netfilter**。


<!-- more -->

## 流程
iptables有5个链:PREROUTING,INPUT,FORWARD,OUTPUT,POSTROUTING,4个表:filter,nat,mangle,raw。（4表5链）  
4个表的优先级由高到低的顺序为:raw-->mangle-->nat-->filter  


![upload successful](/images/iptables_process.png)

* filter：一般的过滤功能
* nat:用于nat功能（端口映射，地址映射等）
* mangle:用于对特定数据包的修改
* raw:优先级最高，设置raw时一般是为了不再让iptables做数据包的链接跟踪处理，提高性能

图片中，**PREROUTING**会有个分叉，系统是根据IP数据包中的`destination ip address`中的IP地址对数据包进行分发。
如果destination ip adress是**本机地址**，数据将会被转交给INPUT链。如果不是本机地址，则交给FORWARD链检测。


## 使用

### 启动、停止和重启iptables

虽然 iptables 并不是一项服务，但在 Linux 中还是可以像服务一样对其状态进行管理。

基于SystemD的系统
```
systemctl start iptables
systemctl stop iptables
systemctl restart iptables
```


### 查看规则
查看iptables防火墙策略
```
sudo iptables -vL 
```
或者带上序号
```
sudo iptables -vL  --line-number //--line-number可以显示规则序号，在删除的时候比较方便 
```
以上命令是查看默认的 FILTER 表，如果你只希望查看特定的表，可以在 -t 参数后跟上要单独查看的表名。例如只查看 NAT 表中的规则，可以使用如下命令：
```
sudo iptables -vL -t nat
```


### 添加规则
命令格式为："iptables [-t 表名] 选项 [链名] [条件] [-j 控制动作]"。
此处列出一些常用的动作：  
* ACCEPT：允许数据包通过。
* DROP：直接丢弃数据包，不给任何回应信息，这时候客户端会感觉自己的请求泥牛入海了，过了超时时间才会有反应。
* REJECT：拒绝数据包通过，必要时会给数据发送端一个响应的信息，客户端刚请求就会收到拒绝的信息。
* SNAT：源地址转换，解决内网用户用同一个公网地址上网的问题。
* MASQUERADE：是SNAT的一种特殊形式，适用于动态的、临时会变的ip上。
* DNAT：目标地址转换。
* REDIRECT：在本机做端口映射。
* LOG：在/var/log/messages文件中记录日志信息，然后将数据包传递给下一条规则，也就是说除了记录以外不对数据包做任何其他操作，仍然让下一条规则去匹配。


拒绝转发来自`192.168.1.10`主机的数据，允许转发来自`192.168.0.0/24`网段的数据
```
iptables -A FORWARD -s 192.168.1.11 -j REJECT 
iptables -A FORWARD -s 192.168.0.0/24 -j ACCEPT
说明：注意要把拒绝的放在前面不然就不起作用了啊。
```
允许本机开放从TCP端口20-1024提供的应用服务。
```
iptables -A INPUT -p tcp --dport 20:1024 -j ACCEPT 
iptables -A OUTPUT -p tcp --sport 20:1024 -j ACCEPT
```
允许转发来自192.168.0.0/24局域网段的DNS解析请求数据包。
```
iptables -A FORWARD -s 192.168.0.0/24 -p udp --dport 53 -j ACCEPT 
iptables -A FORWARD -d 192.168.0.0/24 -p udp --sport 53 -j ACCEPT
```
### DNAT和SNAT

我们要做的DNAT要在进入这个菱形转发区域之前，也就是在**PREROUTING链**中做，例如要把访问`202.103.96.112`的访问转发到`192.168.0.112`上：
```
iptables -t nat -A PREROUTING -d 202.103.96.112 -j DNAT --to-destination 192.168.0.112
```
而SNAT自然是要在数据包流出这台机器之前的最后一个链也就是**POSTROUTING链**来进行操作
```
iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -j SNAT --to-source 58.20.51.66
```
假如当前系统用的是ADSL/3G/4G动态拨号方式，那么每次拨号，出口IP都会改变，SNAT就会有局限性。
```
 iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -o eth0 -j MASQUERADE
```
重点在那个『 MASQUERADE 』！这个设定值就是『IP伪装成为封包出去(-o)的那块装置上的IP』！

不管现在eth0的出口获得了怎样的动态ip，MASQUERADE会自动读取eth0现在的ip地址然后做SNAT出去，这样就实现了很好的动态SNAT地址转换。




### 删除iptables规则
```
iptables -D INPUT 3  //删除input的第3条规则  
  
iptables -t nat -D POSTROUTING 1  //删除nat表中postrouting的第一条规则  
  
iptables -F INPUT   //清空 filter表INPUT所有规则  

iptables -F    //清空所有规则  

iptables -t nat -F POSTROUTING   //清空nat表POSTROUTING所有规则  

```

### 保存和恢复
iptables 命令修改后规则只存在于内存中，但是我们保存当前规则用来恢复
```
iptables-save > /etc/sysconfig/iptables.20180606

iptables-restore < /etc/sysconfig/iptables.20180606
```







参考：
* [iptables之FORWARD转发链](https://blog.51cto.com/linuxcgi/1965296)
* [iptables中DNAT、SNAT和MASQUERADE](https://www.cnblogs.com/liuhongru/p/11422460.html)
* [iptables介绍](https://www.jianshu.com/p/196e57a99a9a)
* [Linux iptables用法与NAT](https://www.cnblogs.com/whych/p/9147900.html)