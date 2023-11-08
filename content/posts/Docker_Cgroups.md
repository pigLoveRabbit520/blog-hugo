---
title: 一步步自己做个Docker之Cgroups
author: Salamander
tags:
  - Docker
  - Cgroups
categories:
  - Docker
date: 2020-04-06 20:30:00
---
![docker logo](/images/docker-logo.png)

本文环境：
* OS：Ubuntu 18.04.4 LTS
* Golang版本：1.12.13


## 什么是Linux Cgroups
**Linux Cgroups**（Control Groups）提供了对一组进程及将来的子进程的资源限制、控制和统计的能力，这些资源包括CPU、内存、存储、网络等。本质上来说，**Cgroups** 是内核附加在程序上的一系列钩子(hook)，通过程序运行时对资源的调度触发相应的钩子以达到资源追踪和限制的目的。  

<!-- more -->



























参考：
* [linux cgroups 简介](https://www.cnblogs.com/sparkdev/p/8296063.html)