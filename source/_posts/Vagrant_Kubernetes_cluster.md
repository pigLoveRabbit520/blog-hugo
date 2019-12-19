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
Vagrant是一个基于Ruby的工具，用于创建和部署虚拟化开发环境。它使用Oracle的开源**VirtualBox**（其实也可以用别的）虚拟化系统，使用Chef创建自动化虚拟环境。
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
Vagrant是基于虚拟机（`VirtualBox`，`VMware`这些）的，所以我们还需要安装`VirtualBox`。在Vagrant官网可以它适配的`VirtualBox`版本
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
`Vagrant`跟docker类似，可以提供一致性环境的，它可以编写`Vagrantfile`（类似`docker-compose.yml`）来定义虚拟机中安装什么软件，环境和配置，它使用ruby语法。`Vagrant`也做了[box源](https://app.vagrantup.com/boxes/search)，类似docker image。  
下面给出一个小栗子感受下，这里使用`ubuntu/xenial64`（Ubuntu 16.06 64位）这个box
```
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  ##### DEFINE VM #####
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://app.vagrantup.com/boxes/search.
  config.vm.box = "ubuntu/xenial64"

  config.vm.hostname = "ubuntu-01"
  config.vm.box_check_update = false

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  config.vm.network "private_network", ip: "192.168.10.50"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"


  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # config.vm.network "forwarded_port", guest: 80, host: 8080
end
```
在Vagrantfile对应的目录下终端键入：`vagrant up`，然后`Vagrant`会帮我们下载`ubuntu/trusty64`这个box，不过在中国下载速度非常慢，在运行`vagrant up`时我们可以看到这个box的下载url，你可以用**迅雷**这些工具直接下载，然后在本地手动添加box
```
$ vagrant box add --name ubuntu/trusty64 /home/lucy/virtualbox.box

$ vagrant box list
ubuntu/xenial64 (virtualbox, 0)
```
启动虚拟机你可能会遇到下面的错误：

![upload successful](/images/virtualbox-error.png)

解决方法是在**BIOS**中将**Intel Virtualization Technology**改为Enable。  
启动虚拟机后，你可以通过`vagrant ssh`进入虚拟机。