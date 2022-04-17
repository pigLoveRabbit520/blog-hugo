title: bochs下启动自己的MBR
author: pigLoveRabbit
tags:
  - OS
categories:
  - OS
date: 2022-04-17 23:12:00
---
# Bochs
Bochs是一个x86硬件平台的开源模拟器。你可以当它是一台虚拟的x86的计算机。


本文环境：  
* OS: Ubuntu 20.04.4 LTS
* Bochs： 2.6.9


## 安装Bochs
这个网上教程很多了，先去[下载](https://sourceforge.net/projects/bochs/files/bochs/)，然后configure：
```
./configure \
--prefix=/your_path/bochs \
--enable-debugger\
--enable-disasm \
--enable-iodebug \
--enable-x86-debugger \
--with-x \
--with-x11
```
这里会出点问题，一般是缺少库了：
```
sudo apt-get install libx11-dev ................. for X11/Xlib.h
sudo apt-get install mesa-common-dev........ for GL/glx.h
sudo apt-get install libglu1-mesa-dev ..... for GL/glu.h
sudo apt-get install libxrandr-dev ........... for X11/extensions/Xrandr.h
sudo apt-get install libxi-dev ................... for X11/extensions/XInput.h
```
configure通过就可以编译安装了：
```
make
sudo make install
```

## 配置
当我们在终端输入bochs后，
Bochs会自己在当前目录顺序寻找以下文件作为默认配置文件：
.bochsrc  
bochsrc  
bochsrc.txt  
bochsrc.bxrc(仅对Windows有效)  
我们可以自己创建一个名为.bochsrc的文件，来指定Bochs配置我们想要的虚拟机。  
```
[~/bochs_fun]$ cat bochsrc

#################################################################
# Bochs的配置文件
# Configuration file for Bochs
#################################################################

# how much memory the emulated machine will have
megs: 32

# filenameof ROM images
romimage:file=/usr/local/share/bochs/BIOS-bochs-latest
vgaromimage:file=/usr/local/share/bochs/VGABIOS-lgpl-latest

# which disk image will be used 这个是启动软盘
floppya:1_44=a.img, status=inserted

# choose the boot disk 确定启动方式
#boot: floppy
boot: disk

# where do we send log messages?
log: bochsout.txt

# disable the mouse
mouse: enabled=0

# enable key mapping ,using US layout as default
keyboard:keymap=/usr/local/share/bochs/keymaps/x11-pc-us.map

# 硬盘设置
ata0: enabled=1, ioaddr1=0x1f0, ioaddr2=0x3f0, irq=14
```

## 测试开机










参考：
* [OS篇-Bochs在Ubuntu下的安装教程](https://mikeygithub.github.io/2021/02/26/os/OS%E7%AF%87-Bochs%E5%9C%A8Ubuntu%E4%B8%8B%E7%9A%84%E5%AE%89%E8%A3%85%E6%95%99%E7%A8%8B/)
* [Ubuntu20.04/18.04安装Bochs2.6.9编译运行GeekOS](https://www.cxybb.com/article/Java_c110/115737121)