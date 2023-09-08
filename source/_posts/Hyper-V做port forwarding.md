title: Blender常用操作
author: pigLoveRabbit
tags:
  - windows
  - Hyper-V
categories:
  - Hyper-V
date: 2023-09-08 21:00:00
---
## 创建 NAT 虚拟网络

要用到一些PowerShell命令  
```
New-VMSwitch -SwitchName "new" -SwitchType Internal

New-NetIPAddress -IPAddress 192.168.0.1 -PrefixLength 24 -InterfaceAlias "vEthernet (new)"

New-NetNat -Name MyNATnetwork -InternalIPInterfaceAddressPrefix 192.168.0.0/24
```

`New-VMSwitch`  新建了一个内部的交换机  
`New-NetIPAddress` 设置了网卡的ip  
`New-NetNat` 设定了子网


## 端口映射
最后利用Add-NetNatStaticMapping命令，映射到宿主机端口：
```
Add-NetNatStaticMapping  -NatName MyNATnetwork  -Protocol TCP  -ExternalIPAddress 0.0.0.0/24  -ExternalPort 5400  -InternalIPAddress 192.168.0.3 -InternalPort 5400
```
