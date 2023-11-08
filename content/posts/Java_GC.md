---
title: Java之GC
author: Salamander
tags:
  - GC
  - Java
categories:
  - Java
date: 2020-04-01 09:00:00
---

![upload successful](/images/gc-java.png)  

## GC
GC就是垃圾回收（`Garbage Collection`），如果你写过C++或者C程序的，你就会知道`new`一个数据后，就需要`delete`它的内存，这就是手动管理内存，但这样如果你粗心点的话，就容易造成内存泄露，所以就有了自动垃圾回收，也就我们这里所讨论的GC。Java的GC会对JVM（Java Virtual Machine）中的内存进行标记，并确定哪些内存需要回收，根据一定的回收策略，自动的回收内存，永不停息（Nerver Stop）的保证JVM中的内存空间，防止出现内存泄露和溢出问题。  
其实GC很早就有了，1960年诞生于MIT的**Lisp**是第一门真正使用内存动态分配和垃圾收集技术的语言。

<!-- more -->


## 内存布局

![upload successful](/images/java-memory-layout.png)  

JVM 在执行 Java 程序的过程中会把它所管理的内存划分为若干个不同的数据区域。有的区域依赖线程的启动和结束而建立和销毁，这样的区域可看作是线程私有的区域（Per-Thread Data Area）；其他的区域则随着虚拟机进程的启动而存在，可以看做是线程间共享的区域。  

具体而言，JVM 的运行时数据区域包括如下：  

* **程序计数器**（Program Counter Register）：线程私有的数据区域，保存当前正在执行的虚拟机指令的地址；
* **Java 虚拟机栈**（Java Virtual Machine Stack）：线程私有的数据区域，每个栈帧中存放当前执行方法的本地变量及返回地址；
* **Java 堆（Heap）**：线程共享的数据区域，对象及数组的分配空间；
* **方法区**（Method Area）：线程共享的数据区域，存放由类加载器加载的类型信息、常量及静态变量；
* **本地方法栈（Native Method Stack）**：用于 Native 方法的方法栈  

**运行时常量池**（Runtime Constant Pool）是方法区的一部分，用于存储编译器生成的常量和引用。一般来说，常量的分配在编译时就能确定，但也不全是，也可以存储在运行时期产生的常量。比如String类的intern（）方法，作用是String类维护了一个常量池，如果调用的字符"hello"已经在常量池中，则直接返回常量池中的地址，否则新建一个常量加入池中，并返回地址。





## GC大致过程

![upload successful](/images/pasted-3.png) 


在JVM的堆内存中，被分为了年轻代（Young Generation）和老年代（Old Generation、持久代（Permanent Generation）。其中年轻代又被分为Eden区和survivor区，而survivor区又被分为大小相等的2个区，分别称为S1区和S2区。持久代在**Sun Hotpot虚拟机**中就是指方法区（有些JVM根本就没有持久代这一说法）

当程序需要在堆上分配内存时，会首先在eden区进行分配。

当eden区内存已满无法再分配对象时，会触发第一次minor gc，将eden区存活的对象拷贝到其中一个survivor区，比如S1，并把eden区的内存清空，以备使用。

当eden区的内存再次被使用完时，会触发第二次minor gc，将eden区和S1区中存活的对象拷贝到S2区，并将eden区和S1区的内存清空以备使用。在有的书中会提到from区和to区，其实from区和to区是相对而言的，当把S1区的内容拷贝到S2区时，S1为from区，S2为to区。而将S2区的内容拷贝到S1时，S2为from区，S1为to区。

当eden区的内存再次被用完时，会触发第三次minor gc，将eden区和S2区存活的对象拷贝到S1区，并清空eden区和S2区的内存，以备使用。

上述过程循环往复。

当某个对象经历了一次minor gc并且存活下来，没有被清理掉，则说这个对象长大了一岁。当某个对象的年龄大于一定阈值，通常是15岁时，该对象在下次minor gc时会被放到老年代中。

当老年代中的内存不够用时，会触发full gc。