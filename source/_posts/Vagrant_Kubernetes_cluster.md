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
  
  config.vm.provider "virtualbox" do |v|
    v.name = "ubuntu-for-fun"
    v.customize ["modifyvm", :id, "--memory", "2048"]
    v.customize ["modifyvm", :id, "--cpus", "2"]
  end

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # config.vm.network "forwarded_port", guest: 80, host: 8080
end
```
更多虚拟机的配置可以查看[官方文档](https://www.vagrantup.com/docs/vagrantfile/machine_settings.html)  
在Vagrantfile对应的目录下终端键入：`vagrant up`，然后`Vagrant`会帮我们下载`ubuntu/xenial64`这个box，不过在中国下载速度非常慢，在运行`vagrant up`时我们可以看到这个box的下载url，你可以用**迅雷**这些工具直接下载，然后在本地手动添加box
```
$ vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Box 'ubuntu/xenial64' could not be found. Attempting to find and install...
    default: Box Provider: virtualbox
    default: Box Version: >= 0
==> default: Loading metadata for box 'ubuntu/xenial64'
    default: URL: https://vagrantcloud.com/ubuntu/xenial64
==> default: Adding box 'ubuntu/xenial64' (v20191217.0.0) for provider: virtualbox
    default: Downloading: https://vagrantcloud.com/ubuntu/boxes/xenial64/versions/20191217.0.0/providers/virtualbox.box
==> default: Box download is resuming from prior download progress
    default: Download redirected to host: cloud-images.ubuntu.com
    .........

$ cd ~/box-add
$ ls
metadata.json  virtualbox.box
$ vagrant box add metadata.json
==> box: Loading metadata for box 'metadata.json'
    box: URL: file:///home/lucy/vm-add/metadata.json
==> box: Adding box 'ubuntu/xenial64' (v20191217.0.0) for provider: virtualbox
    box: Downloading: ./virtualbox.box
==> box: Successfully added box 'ubuntu/xenial64' (v20191217.0.0) for 'virtualbox'!
$ vagrant box list
ubuntu/xenial64 (virtualbox, 20191217.0.0)
```
下载box的URL是`https://vagrantcloud.com/ubuntu/boxes/xenial64/versions/20191217.0.0/providers/virtualbox.box`，可以看到下载的版本是**20191217.0.0**，另外注意一下这里添加box的是使用一个`metadata.json`文件，使用这样的方式可以定义box版本号，它的内容是：
```
{
    "name": "ubuntu/xenial64",
    "versions": [{
        "version": "20191217.0.0",
        "providers": [{
            "name": "virtualbox",
            "url": "./virtualbox.box"
        }]
    }]
}
```

启动虚拟机你可能会遇到下面的错误：

![upload successful](/images/virtualbox-error.png)

解决方法是在**BIOS**中将**Intel Virtualization Technology**改为Enable。  
启动虚拟机后，你可以通过`vagrant ssh`进入虚拟机。

## 启动Kubernetes集群
这里我编写了一个`Vagrantfile`，一键启动集群：
```
# -*- mode: ruby -*-
# vi: set ft=ruby :

servers = [
    {
        :name => "k8s-head",
        :type => "master",
        :box => "ubuntu/xenial64",
        :box_version => "20191217.0.0",
        :eth1 => "192.168.205.10",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-node-1",
        :type => "node",
        :box => "ubuntu/xenial64",
        :box_version => "20191217.0.0",
        :eth1 => "192.168.205.11",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-node-2",
        :type => "node",
        :box => "ubuntu/xenial64",
        :box_version => "20191217.0.0",
        :eth1 => "192.168.205.12",
        :mem => "2048",
        :cpu => "2"
    }
]

# This script to install k8s using kubeadm will get executed after a box is provisioned
$configureBox = <<-SCRIPT
    cp /etc/apt/sources.list /etc/apt/sources.list.bak
    # use Aliyun apt source
    cat > /etc/apt/sources.list<<EOF
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security main restricted universe multiverse
EOF

    export DEBIAN_FRONTEND=noninteractive

    # install docker v17.03
    # reason for not using docker provision is that it always installs latest version of the docker, but kubeadm requires 17.03 or older
    apt-get update
    # step 1: 安装必要的一些系统工具
    apt-get install -y apt-transport-https ca-certificates curl software-properties-common
    # step 2: 安装GPG证书
    curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
    add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"
    apt-get update && apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 17.03 | head -1 | awk '{print $3}')
    # run docker commands as vagrant user (sudo not required)
    usermod -aG docker vagrant
    # 修改docker配置
    sudo bash -c 'cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF'
    sudo systemctl daemon-reload
    sudo systemctl restart docker

    # install kubeadm
    apt-get install -y apt-transport-https curl
    curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -  # aliyun GPG
    cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
    apt-get update
    apt-get install -y kubelet=1.15.7-00 kubeadm=1.15.7-00 kubectl=1.15.7-00
    apt-mark hold kubelet kubeadm kubectl
    # kubelet requires swap off
    swapoff -a
    # keep swap off after reboot
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
    # ip of this box
    IP_ADDR=`ifconfig enp0s8 | grep Mask | awk '{print $2}'| cut -f2 -d:`
    # set node-ip
    sudo sh -c 'echo KUBELET_EXTRA_ARGS= >> /etc/default/kubelet'
    sudo sed -i "/^[^#]*KUBELET_EXTRA_ARGS=/c\KUBELET_EXTRA_ARGS=--node-ip=$IP_ADDR" /etc/default/kubelet
    sudo systemctl restart kubelet
SCRIPT

$configureMaster = <<-SCRIPT
    export DEBIAN_FRONTEND=noninteractive
    echo "This is master"
    # ip of this box
    IP_ADDR=`ifconfig enp0s8 | grep Mask | awk '{print $2}'| cut -f2 -d:`
    # install k8s master
    HOST_NAME=$(hostname -s)
    kubeadm init --image-repository registry.aliyuncs.com/google_containers  --kubernetes-version v1.15.7 \
    --apiserver-advertise-address=$IP_ADDR --apiserver-cert-extra-sans=$IP_ADDR  --node-name $HOST_NAME --pod-network-cidr=172.16.0.0/16
    #copying credentials to regular user - vagrant
    sudo --user=vagrant mkdir -p /home/vagrant/.kube
    cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
    chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

    # install Calico pod network addon
    export KUBECONFIG=/etc/kubernetes/admin.conf
    kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml
    kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
    kubeadm token create --print-join-command >> /etc/kubeadm_join_cmd.sh
    chmod +x /etc/kubeadm_join_cmd.sh
    # required for setting up password less ssh between guest VMs
    sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
    sudo service sshd restart
SCRIPT

$configureNode = <<-SCRIPT
    export DEBIAN_FRONTEND=noninteractive
    echo "This is worker"
    apt-get install -y sshpass
    sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@192.168.205.10:/etc/kubeadm_join_cmd.sh .
    sh ./kubeadm_join_cmd.sh
SCRIPT

Vagrant.configure("2") do |config|
    
    servers.each do |opts|
        config.vm.define opts[:name] do |config|

            config.vm.box = opts[:box]
            config.vm.box_version = opts[:box_version]
            config.vm.hostname = opts[:name]
            config.vm.network :private_network, ip: opts[:eth1]

            config.vm.provider "virtualbox" do |v|

                v.name = opts[:name]
            	v.customize ["modifyvm", :id, "--groups", "/Ballerina Development"]
                v.customize ["modifyvm", :id, "--memory", opts[:mem]]
                v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
                v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
                v.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
            end

            # we cannot use this because we can't install the docker version we want - https://github.com/hashicorp/vagrant/issues/4871
            #config.vm.provision "docker"

            config.vm.provision "shell", inline: $configureBox

            if opts[:type] == "master"
                config.vm.provision "shell", inline: $configureMaster
            else
                config.vm.provision "shell", inline: $configureNode
            end

        end

    end
end
```