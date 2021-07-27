title: Node.js：容器中Core Dump
author: Salamander
tags:
  - nodejs
categories:
  - nodejs
date: 2021-07-27 15:49:00
---
在开始之前，我们先了解下什么是 Core 和 Core Dump。  
**什么是 Core?**  
> 在使用半导体作为内存材料前，人类是利用线圈当作内存的材料，线圈就叫作 core ，用线圈做的内存就叫作 core memory。如今 ，半导体工业澎勃发展，已经没有人用 core memory 了，不过在许多情况下， 人们还是把记忆体叫作 core 。

**什么是 Core Dump?**  

>  当程序运行的过程中异常终止或崩溃，操作系统会将程序当时的内存状态记录下来，保存在一个文件中，这种行为就叫做 Core Dump（中文有的翻译成 “核心转储”)。我们可以认为 Core Dump 是 “内存快照”，但实际上，除了内存信息之外，还有些关键的程序运行状态也会同时 dump 下来，例如寄存器信息（包括程序指针、栈指针等）、内存管理信息、其他处理器和操作系统状态和信息。Core Dump 对于编程人员诊断和调试程序是非常有帮助的，因为对于有些程序错误是很难重现的，例如指针异常，而 Core Dump 文件可以再现程序出错时的情景。  
  

在Node生成core dump文件有两步  
**先开启core dump**  

```
ulimit -c unlimited
```
`ulimit -c`设置允许 Core Dump 生成的文件的大小，如果是 0 则表示关闭了 Core Dump。`unlimited`表示无限制。

**gcore**

第二步我们可以用`gcore`命令（需要装gdb）不重启程序生成core dump文件：
```
gcore [-o filename] pid
```
但在容器中，gcore命令会报错。