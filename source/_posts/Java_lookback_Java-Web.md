title: Java经典回顾之Java Web
author: Salamander
tags:
  - Java
  - Java Web
categories:
  - Java
date: 2020-04-09 21:00:00
---
## Java Web
虽然我们现在会用`SpringBoot`快速创建一个Web Demo，但是基础不能忘（`SpringBoot`或者`SpringMVC`都是封装后的产物），下面就让我们回顾一下一个最基本的Java Web项目。

<!-- more -->

## 创建项目
这里我们使用**IDEA**来创建项目，点击菜单`File`=>`New`=>`Project`，选择`Java Enterprise`，在**Additional Libraries and Framework**中，选择`Web Application`（我这里是4.0，旧版本的`IDEA`可能其他的），`Application Server`就是Java Web项目编译打包运行所需要的Web服务器（你需要自己配置一下）。

![upload successful](/images/idea_java_web.png)

点击`Next`后，填写项目名称就好了，这里我创建了一个`simplejavaweb`的项目。  
我们来看一下**项目结构**  

![upload successful](/images/pasted-4.png)   
* **src**就是我们写Java代码的地方
* **web**目录放了一些web项目的配置