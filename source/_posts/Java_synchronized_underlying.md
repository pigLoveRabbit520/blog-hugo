title: Java之synchronized的实现原理
author: Salamander
tags:
  - synchronized
  - java
categories: []
date: 2020-02-25 19:00:00
---
## synchronized
在Java多线程编程中，我们最先碰到的也是最简单的方法就利用`synchronized`关键字。用它的方式有三种：
* 修饰实例方法，锁是当前实例对象
* 修饰静态方法，锁是当前类的class对象（每个类都有一个Class对象）
* 修饰代码块，锁定括号里的对象

加上`synchronized`之后，我们的代码就变成了同步代码，神奇又强大，但有的时候也不禁会思考下：Java底层是怎么实现`synchronized`关键字的？  
在阅读了一些文章之后，我在这里做了一些归纳和总结。

<!-- more -->

首先，看一段简单的Java代码：
```
public class SyncTest {
    public synchronized void test1(){

    }

    public void test2(){
        synchronized (this){

        }
    }
}
```
先用`javac SyncTest.java`编译出class文件，再利用`javap -v -c SyncTest.class`查看`synchronized`的实现。`javap`是jdk自带的反解析工具。它的作用就是根据class字节码文件，反解析出当前类对应的code区（汇编指令）、本地变量表、异常表和代码行偏移量映射表、常量池等等信息。  
反解析结果为：
```
....
{
  public synchronized void test1();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=0, locals=1, args_size=1
         0: return
      LineNumberTable:
        line 4: 0

  public void test2();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter          // 监视器进入，获取锁
         4: aload_1
         5: monitorexit           // 监视器退出，释放锁
         6: goto          14
         9: astore_2
        10: aload_1
        11: monitorexit
        12: aload_2
        13: athrow
        14: return
      Exception table:
         from    to  target type
             4     6     9   any
             9    12     9   any
      LineNumberTable:
        line 7: 0
        line 9: 4
        line 10: 14
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 9
          locals = [ class SyncTest, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4
}
SourceFile: "SyncTest.java"
```

从上面可以看出，同步代码块是使用monitorenter和monitorexit指令实现的，同步方法（在这看不出来需要看JVM底层实现）依靠的是方法修饰符上的`ACC_SYNCHRONIZED实现`。


## Java对象头
Java对象头和monitor是实现synchronized的基础。  
HotSpot虚拟机中，对象在内存中存储的布局可以分为三块区域：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）。  

![](https://s2.ax1x.com/2020/03/01/3gzdxg.png)  
普通对象的对象头包括两部分：Mark Word 和 Class Metadata Address （类型指针）（如上图所示），如果是数组对象还包括一个额外的Array length数组长度部分（这里我没给出图片）。  

在32位虚拟机中，一字宽等于四字节，即32bit。  
HotSpot虚拟机的对象头(Object Header)包括两部分信息，第一部分用于存储对象自身的运行时数据， 如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等等，这部分数据的长度在32位和64位的虚拟机（暂不考虑开启压缩指针的场景）中分别为32个和64个Bits，官方称它为“Mark Word”（标记字段）。  

对象需要存储的运行时数据很多，其实已经超出了32、64位Bitmap结构所能记录的限度，但是对象头信息是与对象自身定义的数据无关的额 外存储成本，考虑到虚拟机的空间效率，Mark Word被设计成一个非固定的数据结构以便在极小的空间内存储尽量多的信息，它会根据对象的状态复用自己的存储空间。例如在32位的HotSpot虚拟机 中对象未被锁定的状态下，Mark Word的32个Bits空间中的25Bits用于存储对象哈希码（HashCode），4Bits用于存储对象分代年龄，2Bits用于存储锁标志 位，1Bit固定为0，在其他状态（轻量级锁定、重量级锁定、GC标记、可偏向）下对象的存储内容如下表所示：  

![](https://s2.ax1x.com/2020/03/01/32ClfP.jpg)

注意偏向锁、轻量级锁、重量级锁等都是jdk 1.6以后引入的。  
![](https://s2.ax1x.com/2020/03/01/32iYIs.png)

## monitor对象

轻量级锁和偏向锁是Java 6 对 synchronized 锁进行优化后新增加的，稍后我们会简要分析。这里我们主要分析一下重量级锁也就是通常说`synchronized`的对象锁，锁标识位为10，其中指针指向的是monitor对象（也称为管程或监视器锁）的起始地址。每个对象都存在着一个 monitor 与之关联，对象与其 monitor 之间的关系有存在多种实现方式，如monitor可以与对象一起创建销毁或当线程试图获取对象锁时自动生成，但当一个 monitor 被某个线程持有后，它便处于锁定状态。在Java虚拟机(HotSpot)中，monitor是由ObjectMonitor实现的，其主要数据结构如下（位于HotSpot虚拟机源码ObjectMonitor.hpp文件，C++实现的）：  
```
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //记录个数
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```

ObjectMonitor中有两个队列，_WaitSet和_EntryList，用来保存ObjectWaiter对象列表(每个等待锁的线程都会被封装成ObjectWaiter对象)，_owner指向持有ObjectMonitor对象的线程，当多个线程同时访问一段同步代码时，首先会进入 _EntryList 集合，当线程获取到对象的monitor 后进入 _Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1，若线程调用wait()方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入WaitSet集合中等待被唤醒。若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)。如下图所示：  

![](https://s2.ax1x.com/2020/03/06/3qfHBQ.gif)





















参考：
* [【死磕Java并发】-----深入分析synchronized的实现原理
](https://blog.csdn.net/chenssy/article/details/54883355)
* [java对象在内存中的结构（HotSpot虚拟机）
](https://www.cnblogs.com/duanxz/p/4967042.html)
* [Difference between lock and monitor – Java Concurrency
](https://howtodoinjava.com/java/multi-threading/multithreading-difference-between-lock-and-monitor/)