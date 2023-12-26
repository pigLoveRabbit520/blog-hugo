---
title: wsl2 install CUDA
author: pigLoveRabbit
tags:
  - cuda
  - wsl2
categories:
  - wsl2
date: 2023-12-10 16:00:00
---
##### 驱动
网上有许多资料写道需要安装针对WSL特别驱动，但是现在已经不需要这么做了，只需要到NVIDIA官网将[驱动](https://www.nvidia.com/Download/index.aspx)升级到最新版本即可。根据参考资料描述，驱动类型最好选择Game Ready版本而不是studio版本。  
![image](/images/nvidia_download.jpg)
注意，该是安装Windows驱动，而不是安装Linux驱动，在Windows下安装驱动后，会自动将驱动以_libcuda.so_的形式集成至WSL2中，因此**切勿在WSL Linux中重复安装驱动**。

##### CUDA
这一步要小心，WSL2中安装CUDA和在普通Linux中安装CUDA会有所不同，要选择`WSL-Ubuntu`类型
![image](/images/cuda_wsl2.jpg)

然后按命令安装CUDA
```bash
wget https://developer.download.nvidia.com/compute/cuda/12.3.1/local_installers/cuda_12.3.1_545.23.08_linux.run
sudo sh cuda_12.3.1_545.23.08_linux.run
```
最后`nvcc`命令还是没找到，得`vim ~/.bashrc`，把cuda-toolkit的东西加到环境变量里（官方脚本没写好吧^=^）
```
export PATH=${PATH}:/usr/local/cuda/bin
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/usr/local/cuda/lib64
```

最后，看一下`nvcc`命令  
```
$ nvcc --version
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2023 NVIDIA Corporation
Built on Fri_Nov__3_17:16:49_PDT_2023
Cuda compilation tools, release 12.3, V12.3.103
Build cuda_12.3.r12.3/compiler.33492891_0
```