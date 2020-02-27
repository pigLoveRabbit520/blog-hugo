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

















参考：
* [【死磕Java并发】-----深入分析synchronized的实现原理
](https://blog.csdn.net/chenssy/article/details/54883355)