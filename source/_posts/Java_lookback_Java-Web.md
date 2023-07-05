title: Java经典回顾之Java Web
author: Salamander
tags:
  - Java
  - Java Web
categories:
  - Java
date: 2020-04-09 21:00:00
---
![upload successful](/images/java-web-develop.jpg)

本文环境：
* OS：Ubuntu 18.04.4 LTS
* Java版本：1.8.0_221


## Java Web
虽然我们现在会用`SpringBoot`快速创建一个Web Demo，但是基础不能忘（`SpringBoot`或者`SpringMVC`都是封装后的产物），下面就让我们回顾一下一个最基本的Java Web项目。

<!-- more -->

## 创建项目
这里我们使用**IDEA**来创建项目，点击菜单`File`=>`New`=>`Project`，选择`Java Enterprise`，在**Additional Libraries and Framework**中，选择`Web Application`（我这里是4.0，旧版本的`IDEA`可能其他的），`Application Server`就是Java Web项目编译打包后运行所需要的Web服务器（你需要自己配置一下）。

![upload successful](/images/idea_java_web.png)

点击`Next`后，填写项目名称就好了，这里我创建了一个`simplejavaweb`的项目。  
我们来看一下**项目结构**  

![upload successful](/images/pasted-4.png)   
* **src**就是我们写Java代码的地方
* **web**目录是web应用部署根目录
* web中的**WEB_INF**是Java的web应用的安全目录。所谓安全就是客户端无法访问，只有服务端可以访问的目录。如果想在页面中直接访问其中的文件，必须通过web.xml文件对要访问的文件进行相应映射才能访问。  
* WEB_INF中的**web.xml**是Java web 项目最主要的构成部分之一，它是Web应用程序配置文件，描述了 `servlet` 和其他的应用组件配置及命名规则。