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

<!-- more -->

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

## 信号和槽
信号和槽机制是 QT 的**核心机制**，要精通 QT 编程就必须对信号和槽有所了解。不同于传统的函数回调方式。信号和插槽是 Qt 中非常有特色的地方，可以说是Qt编程区别于其它编程的标志。信号和槽是一种高级接口，应用于对象之间的通信，它是 Qt 的核心特性。

### 信号（signal）
当一个对象中某些可能会有别的对象关心的状态被修改时，将会发出信号。只有定义了信号的类及其子类可以发出信号。

当一个信号被发出时，连接到这个信号的槽立即被调用，就像一个普通的函数调用。当这种情况发生时，信号槽机制独立于任何 GUI 事件循环。emit 语句之后的代码将在所有的槽返回之后被执行。这种情况与使用连接队列略有不同：使用连接队列的时候，emit 语句之后的代码将立即被执行，而槽在之后执行。

如果一个信号连接了多个槽，当信号发出时，这些槽将以连接的顺序一个接一个地被执行（顺序不确定）。


### 槽（slot）
当连接到的信号发出时，槽就会被调用。槽是**普通的 C++ 函数**，能够被正常的调用。它们的唯一特点是能够与信号连接。

既然信号就是普通的成员函数，当它们像普通函数一样调用的时候，遵循标准 C++ 的规则。但是，作为槽，它们又能够通过信号槽的连接被任何组件调用，不论这个组件的访问级别。这意味着任意类的实例发出的信号，都可以使得不相关的类的私有槽被调用。  

你也能把槽定义成虚的，这一点在实际应用中非常有用。

connect()语句的原型类似于：
```
connect(sender, SIGNAL(signal), receiver, SLOT(slot));
```
这里，sender 和 receiver 都是 **QObject** 类型的，singal 和 slot 都是没有参数名称的函数签名。SINGAL()和SLOT()宏用于把参数转换成字符串。  
一个信号可以和多个槽相连：
```
connect(slider, SIGNAL(valueChanged(int)),
              spinBox, SLOT(setValue(int))); 
connect(slider, SIGNAL(valueChanged(int)),
              this, SLOT(updateStatusBarIndicator(int)));
```












## 参考

* 油管上VoidRealms的[Qt视频](https://www.youtube.com/watch?v=Id-sPu_m_hE&t=176s)