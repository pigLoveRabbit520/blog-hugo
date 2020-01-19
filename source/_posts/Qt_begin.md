title: Qt之初步尝试
author: Salamander
tags:
  - Qt
  - Qt Creator
categories:
  - C++
date: 2020-01-18 10:00:00
---
本文环境：
* OS：Ubuntu 18.04.3 LTS
* Qt版本：5.13.1
* Qt Creator版本：4.10.1

## Qt安装
首先，我们得明白一些概念。  
**Qt**是一个C++库，或者说是开发框架，里面集成了一些库函数，提高开发效率。  
**Qt Creator**是Qt集成开发环境，你可以在这里编写，编译，运行你的程序。所以最开始写Qt只安装**Qt Creator**这个是不行的，因为还没有相关的Qt库呢，但是新版的**Qt Creator**（5.9开始）已经集成了Qt了，所以入门就方便很多了。  
关于Qt下载，大家可以打开这里的[链接](http://download.qt.io/archive/qt/)，里面有各版本Qt（**Qt**和**Qt Creator**的集成包），操作简单，最新版本是**5.14**。  
windows版本只要双击exe就可以安装了，Linux版本需要先添加执行权限然后运行文件
```
$ chmod +x qt-opensource-linux-x64-5.13.2.run
$ ./qt-opensource-linux-x64-5.13.2.run
```
对于Linux系统，需要安装C/C++编译器，以Ubuntu为例，需要执行：
```
sudo apt-get install -y gcc g++
```

这一步需要注册一个账号，随便注册一个即可。  
![install 1](https://s2.ax1x.com/2020/01/18/1p7NOs.png)  
这一步选择你需要的组件（不清楚的话，就像我这样选择好了）  
![install 2](https://s2.ax1x.com/2020/01/18/1p7IfO.png)  
最后来到Qt Creator的启动界面  
![](https://s2.ax1x.com/2020/01/18/1pqSyQ.png)

## 写个hello world
点击**文件**菜单，然后新建项目，选择`Qt Console Application`。  
![](https://s2.ax1x.com/2020/01/18/19CDSA.png)  
编辑`main.cpp`文件，代码为：  
```
#include <QCoreApplication>
#include <QDebug>

int main(int argc, char *argv[])
{
    QCoreApplication a(argc, argv);

    qDebug() << "hello world";
    return a.exec();
}
```
点击左下角的`Run`按钮，就可以启动程序。  
![](https://s2.ax1x.com/2020/01/18/19PDNF.png) 