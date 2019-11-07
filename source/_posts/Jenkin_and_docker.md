title: Jenkins在Docker中运行中的坑
author: Salamander
tags:
  - jenkins
  - ci
  - docker
categories:
  - ci
date: 2019-11-07 20:00:00
---

## jenkins是什么？
  Jenkins是一个开源的、提供友好操作界面的持续集成(CI)工具，起源于Hudson（Hudson是商用的），主要用于持续、自动的构建/测试软件项目、监控外部任务的运行。Jenkins用Java语言编写，可在Tomcat等流行的servlet容器中运行，也可独立运行。通常与版本管理工具(SCM)、构建工具结合使用。常用的版本控制工具有SVN、GIT，构建工具有Maven、Ant、Gradle。  
上面的介绍是抄的（逃，简单讲，就是Jenkins能帮我们**自动编译，测试，发布软件**。

## 安装运行
Jenkins有单独的war包，通过`java -jar jenkins.war`直接就可以运行（[官网下载](https://jenkins.io/zh/download/)，选择`Generic Java package (.war)`，或者[官方镜像](http://mirrors.jenkins.io/)），选择LTS Releases	中的`war-stable`），但是jre环境，当然对于熟悉Java的人来说，这个是配置一下即可。本文介绍在Docker中运行Jenkins以及会遇到的一些问题。  
* 操作系统：Ubuntu 18.04.3 LTS
* docker版本：19.03.4
* jdk版本：java version "1.8.0_221"

在`vim`中打开中文有时候会乱码，可以通过下面命令解决：
```
sudo locale-gen zh_CN.UTF-8
```
好了，让我们开始安装`Jenkins`。  
首先，编写一份自定义的`Dockerfile`：
```
FROM jenkins/jenkins:lts

USER root

RUN echo ' \n\
deb http://mirrors.aliyun.com/debian stretch main contrib non-free \n\
deb-src http://mirrors.aliyun.com/debian stretch main contrib non-free \n\
deb http://mirrors.aliyun.com/debian stretch-updates main contrib non-free \n\
deb-src http://mirrors.aliyun.com/debian stretch-updates main contrib non-free \n\
deb http://mirrors.aliyun.com/debian-security stretch/updates main contrib non-free \n\
deb-src http://mirrors.aliyun.com/debian-security stretch/updates main contrib non-free ' > /etc/apt/sources.list

RUN cat /etc/apt/sources.list

#更新源并安装缺少的包
RUN apt-get update && apt-get install -y gcc g++ make openssl pkg-config


USER jenkins
```
基础镜像是`jenkins/jenkins:lts`，观察一下这个镜像
![](https://s2.ax1x.com/2019/11/07/MAz11g.png)  
发现它是基于`FROM openjdk:8-jdk-stretch`，这是带有jdk的debian 9镜像。所以我在`Dockerfile`中修改了apt源，这里使用了阿里云的apt源（`\n\`是换行加上续行符）。

再配合`docker-compose.yml`：
```
version: '3'
services:
  jenkins:
    build: .
    volumes:
      - ./data:/var/jenkins_home
    environment:
      - "JAVA_OPTS=-Duser.timezone=Asia/Shanghai -Xms1g -Xmx1g"
    ports: 
      - 127.0.0.1:8080:8080
      - 50000:50000
```