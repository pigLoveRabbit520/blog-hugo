title: Ubuntu上安装NVIDIA显卡驱动和CUDA和cuDNN库
author: Salamander
tags:
  - NVI
  - CUDA
  - cuDNN
categories:
  - 机器学习
date: 2019-09-18 16:00:00
---
最近需要在用[Pytorch](https://pytorch.org/)做深度学习，为了加快训练速度，需要用到GPU运算，故在此记录一下安装过程。  
我的本机环境：
* Ubuntu 18.04.3 LTS
* GeForce RTX 2080s

### 检查BIOS启动项
* 在开机启动项的Security选项中检查UEFI是否开启，如果开启的话请立马关掉它（重要）  
* 在开机启动项的Boot选项中检查Secure Boot是否开启，如果开启的话请立马关掉它（重要），对于有的BIOS，只要删除**Secure Boot Key**就好了。

<!-- more -->

### 禁用 nouveau
运行命令
```
 sudo gedit /etc/modprobe.d/blacklist.conf
```
将下列代码增加到blacklist.conf文件的末尾：
```
blacklist vga16fb

blacklist nouveau

blacklist rivafb

blacklist rivatv

blacklist nvidiafb
```
保存，然后在命令行中更新initramfs，运行：
```
 sudo update-initramfs -u
```
之后，重启主机
```
reboot
```
在终端运行，运行以下命令，查看是否禁用nouveau成功（无输出则表示禁用成功）：
```
lsmod | grep nouveau
```


### 安装显卡驱动
在NVIDIA官方选择对应驱动，然后[下载](https://www.geforce.com/drivers)：

![图片](https://s2.ax1x.com/2019/09/18/n7DK2Q.png)

在安装驱动之前，应该卸载原有的NVIDIA驱动程序
```
sudo apt-get remove –purge nvidia*
```
把下载的驱动放到用户目录下，我这里下载文件为`NVIDIA-Linux-x86_64-430.50.run`  
为了安装新的NVIDIA驱动程序，我们需要停止当前的显示服务器。最简单的方法是使用telinit命令更改为运行级别3。执行以下linux命令后，显示服务器将停止，因此请确保在继续之前保存所有当前工作（如果有）：
```
sudo telinit 3
```
之后会进入一个新的命令行会话，使用当前的用户名密码登录，然后授予驱动文件可执行权限
```
chmod a+x NVIDIA-Linux-x86_64-430.50.run
```
然后执行安装：
```
sudo ./NVIDIA-Linux-x86_64-430.50.run --no-opengl-files
```
注意，**–no-opengl-files**参数必须加否则会循环登录，也就是loop login  
参数介绍：
* –no-opengl-files 只安装驱动文件，不安装OpenGL文件。这个参数最重要
* –no-x-check 安装驱动时不检查X服务
* –no-nouveau-check 安装驱动时不检查nouveau

后面两个参数可不加。  


安装驱动中注意，**pre-install script failed**这个提示没什么关系，之后的warning提示**unable to find a suitable destination to install 32-bit compatibility libraries**也没关系，都选ok，在询问是否修改x-configuration，请选择默认的**no**，选择yes会导致重启后无法进入系统。


### 使用nvidia-smi命令测试
英伟达系统管理接口（NVIDIA System Management Interface, 简称 nvidia-smi）是基于NVIDIA Management Library (NVML) 的命令行管理组件,旨在(intened to )帮助管理和监控NVIDIA GPU设备。

驱动安装完成后，启动电脑，之后就能用nvidia-smi命令判断驱动是否安装成功
```
nvidia-smi
```
执行这条命令将会打印出当前系统安装的NVIDIA驱动信息，如下：

![image](https://s2.ax1x.com/2019/09/18/n7y6I0.png)

若出现上图中的结果则说明英伟达驱动安装成功。



### 安装CUDA10.1
CUDA是什么？  
>> CUDA，Compute Unified Device Architecture的简称，是由NVIDIA公司创立的基于他们公司生产的图形处理器GPUs（Graphics Processing Units,可以通俗的理解为显卡）的一个并行计算平台和编程模型。
        通过CUDA，GPUs可以很方便地被用来进行通用计算（有点像在CPU中进行的数值计算等等）。在没有CUDA之前，GPUs一般只用来进行图形渲染（如通过OpenGL，DirectX）。
        
   

下载[地址](https://developer.nvidia.com/cuda-downloads)，选择对应版本的cuda安装包，我这里选择的是`runfile`类型的，不要选择使用`deb`版本，**安装CUDA时一定使用runfile文件，这样可以进行选择不再安装驱动**。

![img](https://s2.ax1x.com/2019/09/18/n76jhV.png)

在安装界面，**注意选择不安装显卡驱动**（按enter键取消选择）

![img](https://s2.ax1x.com/2019/09/18/n7gC28.png)

。之后，打开/usr/local文件夹，我们会发现多了cuda和cuda10.1这两个文件夹，如下所示：

![img](https://s2.ax1x.com/2019/09/18/n7goZj.png)


### 添加环境变量
运行`sudo vim /etc/profile`，末尾加上：
```
export CUDA_HOME=/usr/local/cuda 
export PATH=$PATH:$CUDA_HOME/bin 
export LD_LIBRARY_PATH=/usr/local/cuda-10.1/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
```
之后运行`source /etc/profile`使变量起效。

### 判断CUDA安装成功
运行一下代码
```
cd /usr/local/cuda/samples/1_Utilities/deviceQuery 
sudo make
./deviceQuery
```
如果输出如下类似信息，说明CUDA安装成功：

![img](https://s2.ax1x.com/2019/09/18/n72x9P.png)

在CUDA安装之后，我们其实已经可以用PyTorch判断是否支持GPU了，进入python控制台：
```
import torch
print(torch.cuda.is_available())
```

### CUDA与cuDNN的关系
cuDNN是GPU加速计算深层神经网络的库。把CUDA看作是一个工作台，上面配有很多工具，如锤子、螺丝刀等。cuDNN是基于CUDA的深度学习GPU加速库，有了它才能在GPU上完成深度学习的计算。它就相当于工作的工具，比如它就是个扳手。但是CUDA这个工作台买来的时候，并没有送扳手。想要在CUDA上运行深度神经网络，就要安装cuDNN，就像你想要拧个螺帽就要把扳手买回来。这样才能使GPU进行深度神经网络的工作，工作速度相较CPU快很多。



### 安装cuDNN
[官方安装cuDNN指南](https://docs.nvidia.com/deeplearning/sdk/cudnn-install/index.html#install-linux)  
从官方安装指南可以看出，只要把**cuDNN文件复制到CUDA的对应文件夹**里就可以，即是所谓插入式设计，把cuDNN数据库添加CUDA里，cuDNN是CUDA的扩展计算库，不会对CUDA造成其他影响。
![官方安装cuDNN指南](https://s2.ax1x.com/2019/09/20/njMwDK.png)

首先去[官网](https://developer.nvidia.com/rdp/cudnn-archive)下载cuDNN，需要注册一个账号才能下载。注意要选择对应版本的**cuDNN Library for Linux**（与CUDA 10.1对应）： 

![img](https://s2.ax1x.com/2019/09/18/n7W98x.png)
下载后进行解压：
```
tar -zxvf cudnn-10.1-linux-x64-v7.6.2.24.tgz
```
进入cudnn 10.1解压之后的include目录，在命令行进行如下操作：
```
cd cuda/include
sudo cp cudnn.h /usr/local/cuda/include  #复制头文件
```
再将进入lib64目录下的动态文件进行复制和链接：
```
cd ..
cd lib64
sudo cp libcudnn* /usr/local/cuda/lib64/    #复制动态链接库
cd /usr/local/cuda/lib64/
sudo chmod +r libcudnn.so.7.6.2
sudo ln -sf libcudnn.so.7.6.2 libcudnn.so.7
sudo ln -sf libcudnn.so.7 libcudnn.so
sudo chmod a+r /usr/local/cuda/include/cudnn.h /usr/local/cuda/lib64/libcudnn*
sudo ldconfig
```


参考文章：
* https://blog.csdn.net/oTengYue/article/details/79506758
* https://shomy.top/2016/12/29/gpu-tensorflow-install
* [简书——CUDA与cuDNN](https://www.jianshu.com/p/622f47f94784)