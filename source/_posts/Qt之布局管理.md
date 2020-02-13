title: Qt之布局管理
author: Salamander
tags:
  - Qt
categories:
  - C++
date: 2020-02-12 21:00:00
---
本文环境：
* OS：Ubuntu 18.04.3 LTS
* Qt版本：5.13.1
* Qt Creator版本：4.10.1


## 布局器概览
我们以下图的 Qt 设计师界面来说明布局功能，QtCreator 设计模式的布局功能与 Qt 设计师是一样的。

![upload successful](/images/pasted-0.png)  

<!-- more -->

在设计师左边列表，可以看到 Layouts 栏目里有四个布局器：
直布局器 QVBoxLayout：将内部的控件按照垂直方向排布，一行一个。  
◆  水平布局器 QHBoxLayout：将内部的控件按照水平方向排布，一列一个。  
◆  网格布局器 QGridLayout：按照多行、多列的网格排布内部控件，单个控件可以占一个格子或者占据连续多个格子。  
◆  表单布局器 QFormLayout：Qt 设计师里把这个布局器称为窗体布局器，窗体布局器这个叫法不准。这个布局器就是对应网页设计的表单，通常用于接收用户输入。该布局器就如它的图标一样，就是固定的两列控 件，第一列通常是标签，第二列是输入控件或含有输入控件的布局器。  
◆  Qt 另外还有一个堆栈布局器 **QStackedLayout**，通常用于容纳多个子窗口布局，每次只显示其中一个。这个布局器隐含在堆栈部件 QStackedWidget 内部，一般直接用 QStackedWidget 就行了，不需要专门设置堆栈布局器。    

与布局紧密关联的是两个空白条（或叫弹簧条）：**Horizontal Spacer** 水平空白条和 **Vertical Spacer** 垂直空白条，空白条的作用就是填充无用的空隙，如果不希望看到控件拉伸后变丑，就可以塞一个空白条到布局器里面，布局器通常会优先拉伸空白条。两种空白条的类名都是 QSpacerItem，两种空白条只是默认的拉伸方向不一样。


## QBoxLayout
水平布局器 QHBoxLayout 和垂直布局器 QVBoxLayout 的基类都是 QBoxLayout，只是二者排列方向不同。水平和垂直布局器的主要功能函数都位于基类 QBoxLayout 里面，我们这里专门介绍一下这个基类的功能。  
QBoxLayout 构造函数和 setDirection() 都可以指定布局器的方向：
```
QBoxLayout(Direction dir, QWidget * parent = 0)
void setDirection(Direction direction)
```
QBoxLayout 布局器的方向 QBoxLayout::​Direction 枚举不仅可以指定水平和垂直，还能指定反方向排列：  


| 枚举常量                    | 数值  | 描述            |
|-------------------------|-----|---------------|
| QBoxLayout::LeftToRight |  0  |  水平布局，从左到右排列  |
| QBoxLayout::RightToLeft |  1  |  水平布局，从右到左排列  |
| QBoxLayout::TopToBottom |  2  |  垂直布局，从上到下排列  |
| QBoxLayout::BottomToTop |  3  |  垂直布局，从下到上排列  |

水平布局器 QHBoxLayout 和垂直布局器 QVBoxLayout 默认是其中的两种：**QBoxLayout::LeftToRight** 和 **QBoxLayout::TopToBottom** 。  

布局器是一定要往里面添加控件才有用，添加控件的函数如下：
```
void addWidget(QWidget * widget, int stretch = 0, Qt::Alignment alignment = 0)
void insertWidget(int index, QWidget * widget, int stretch = 0, Qt::Alignment alignment = 0)
```
widget 就是要添加的控件指针，**stretch** 是伸展因子，伸展因子越大，窗口变大时拉伸越 多，**alignment** 一般不需要指定，用默认的即可。  
第一个 **addWidget()** 是将控件添加到布局里面的控件列表末尾，第二个 **insertWidget()** 是将控件插入到布局里控件列表序号为 index 的位置。

下面看个例子，我在垂直布局器中添加了5个Label，它们高度按不同的比例分配  
**mainwwindow.cpp**  
```
MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
    , ui(new Ui::MainWindow)
{
    ui->setupUi(this);

    QLabel *label1 = new QLabel("One");
    QLabel *label2 = new QLabel("Two");
    QLabel *label3 = new QLabel("Three");
    QLabel *label4 = new QLabel("Four");
    QLabel *label5 = new QLabel("Five");

    label1->setStyleSheet("background-color: red");
    label2->setStyleSheet("background-color: yellow");
    label3->setStyleSheet("background-color: green");
    label4->setStyleSheet("background-color: black");
    label5->setStyleSheet("background-color: orange");

    QVBoxLayout *layout = new QVBoxLayout;
    layout->addWidget(label1);
    layout->addWidget(label2, 2);
    layout->addWidget(label3, 3);
    layout->addWidget(label4, 4);
    layout->addWidget(label5, 5);

    auto central = new QWidget;
    central->setLayout(layout);

    this->setCentralWidget(central);
}
```
最终呈现的效果是：  
![](https://s2.ax1x.com/2020/02/12/1b3US0.png)  
然后，我再添加一个水品布局器，在里头放入3个label  
```
QHBoxLayout *layoutH = new QHBoxLayout;
layout->addLayout(layoutH, 4);  // stretch比例为4
QLabel *label6 = new QLabel("Six");
QLabel *label7 = new QLabel("seven");
QLabel *label8 = new QLabel("eight");

label6->setStyleSheet("background-color: #7B72E9");
label7->setStyleSheet("background-color: #1B9AF7");
label8->setStyleSheet("background-color: #FF4351");

layoutH->addWidget(label6);
layoutH->addWidget(label7);
layoutH->addWidget(label8);

```
最终效果为：  

![upload successful](/images/pasted-1.png)


参考：  
* [6.2 水平和垂直布局器](https://qtguide.ustclug.org/ch06-02.htm)