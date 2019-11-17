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

<!-- more -->


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
现在我们就可以启动Jenkins了，打开终端，键入命令：
```
docker-compose up
```
这时候，我们会遇到错误：
```
jenkins_1  | touch: cannot touch '/var/jenkins_home/copy_reference_file.log': Permission denied
jenkins_1  | Can not write to /var/jenkins_home/copy_reference_file.log. Wrong volume permissions?
```
看描述是**权限问题**，观察一下目录下的`data`文件夹：
```
drwxr-xr-x  2 root       root       4096 11月 17 20:47 data
```
发现目录的属主是`root`用户，这是什么原因呢？

## 原因探究
查看`Jenkins`容器的当前用户和目录`/var/jenkins_home`属主，我们发现当前用户是`Jenkins`，`/var/jenkins_home`属主用户是`jenkins`：
```
docker run -ti --rm --entrypoint="/bin/bash"  jenkins/jenkins:lts  -c "whoami && id"
jenkins
uid=1000(jenkins) gid=1000(jenkins) groups=1000(jenkins)

docker run -ti --rm --entrypoint="/bin/bash" jenkins/jenkins:lts -c "ls -la /var"
drwxr-xr-x 1 root    root    4096 Oct 17 08:29 cache
drwxr-xr-x 2 jenkins jenkins 4096 Nov 17 14:05 jenkins_home

```
上述命令中，`--rm`选项是让容器退出时自动清除，`--entrypoint`是覆盖镜像中的`ENTRYPOINT`。  
现在我们知道了，因为`/var/jenkins_home`映射到本地数据卷时，目录的拥有者变成了root用户，所以出现了`Permission denied`的问题。  
发现问题之后，相应的解决方法也很简单：把当前目录的拥有者赋值给uid 1000，再启动"jenkins"容器就一切正常了。
```
sudo chown -R 1000:1000 data
```
这时利用浏览器访问 "http://localhost:8080/" 就可以看到Jenkins的经典Web界面了。




参考：
* [谈谈 Docker Volume 之权限管理（一）](https://yq.aliyun.com/articles/53990)