title: Vagrant本地快速启动Kubernetes集群
author: Salamander
tags:
  - kubernetes
  - vagrant
  - virtualbox
categories:
  - kubernetes
date: 2019-12-16 13:00:00
---
![k8s logo](https://image-static.segmentfault.com/311/703/311703680-5b80e2877f8c8_articlex)

Kubernetes，简称 **k8s**（k，8 个字符，s——明白了？）或者 “kube”，是一个开源的 Linux 容器自动化运维平台，它消除了容器化应用程序在部署、伸缩时涉及到的许多手动操作。换句话说，你可以将多台主机组合成集群来运行 Linux 容器，而 Kubernetes 可以帮助你简单高效地管理那些集群。构成这些集群的主机还可以跨越公有云、私有云以及混合云。



本文环境：
* OS：Ubuntu 18.04.3 LTS
* Vagrant版本：2.2.6
* VirtualBox版本：6.0.14 r133895 (Qt5.9.5)

<!-- more -->

## 安装Vagrant
Vagrant是一个基于Ruby的工具，用于创建和部署虚拟化开发环境。它使用Oracle的开源**VirtualBox**虚拟化系统，使用Chef创建自动化虚拟环境。
首先到[官网](https://www.vagrantup.com/downloads.html)下载最新的`Vagrant`，现在最新的版本是**2.2.6**，当然你也可以通过命令行下载：
```
wget https://releases.hashicorp.com/vagrant/2.2.6/vagrant_2.2.6_x86_64.deb
```
验证`Vagrant`安装成功
```
$ vagrant --version
Vagrant 2.2.6
```

## 安装VirtualBox
Vagrant是基于`VirtualBox`的，所以我们还需要安装`VirtualBox`。在Vagrant官网可以它适配的`VirtualBox`版本
> Vagrant comes with support out of the box for VirtualBox, a free, cross-platform consumer virtualization product.
> The VirtualBox provider is compatible with VirtualBox versions 4.0.x, 4.1.x, 4.2.x, 4.3.x, 5.0.x, 5.1.x, 5.2.x, and 6.0.x.

这里我下载6.0版本的`VirtualBox`，[下载地址](https://www.virtualbox.org/wiki/Download_Old_Builds_6_0)
```
wget https://download.virtualbox.org/virtualbox/6.0.14/virtualbox-6.0_6.0.14-133895~Ubuntu~bionic_amd64.deb
```
**注意：不要通过apt-get安装VirtualBox**，因为5.1.0版本开始，VirtualBox已经不需要**DKMS**，apt官方源中VirtualBox比较老，是会带上`DKMS`的：
```
DKMS isn't required by VirtualBox since 5.1.0. Which means that you downloaded VirtualBox from your Debian "store". That's a fork, not supported. You can either ask in their forums for help, or completely remove/uninstall/delete/purge their version and install the official version from the Downloads section of VirtualBox (https://www.virtualbox.org/wiki/Downloads).
```


## 启动虚拟机