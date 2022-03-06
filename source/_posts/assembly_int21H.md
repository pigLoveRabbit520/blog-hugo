title: DOS 系统功能调用（INT 21H）
author: Salamander
tags:
  - Assembly
categories: []
date: 2022-03-05 20:17:00
---
![upload successful](/images/dos_image.png)

`INT`表示`interrupt`（中断）， INT指令是X86汇编语言中最重要的指令之一，它的作用是引发中断，调用“中断例程”（interrupt routine）。   
中断是由于软件或硬件的信号，使CPU暂停执行当前的任务，转而去执行另一段子程序。  
* 硬中断（外中断）：由外部设备，如网卡、硬盘随机引发的，比如网卡收到数据包的时候，就会发出一个中断。
* 软中断（内中断）：由执行中的指令产生的，可以通过程序控制触发。

可以通过 “ INT 中断码 ” 实现中断，内存中有一张**中断向量表**，用来存放中断码处理中断程序的入口地址。CPU在接受到中断信号后，暂停当前正在执行的程序，跳转到中断码对应的向量表地址处去执行中断。


我们理解上可以当`INT`就是调用系统内置的一些功能。  
常用的中断：  
* `INT 21H`：DOS系统功能调用
* `INT 10H`：BIOS终端调用
* `INT 3H`：断点中断，用于调试程序

这篇文章重点记录下DOS的系统功能调用，也就是`INT 21H`。  

<!-- more -->


本文环境：
* 操作系统：虚拟机中的Windows 10
* Visual Studio Code
* VSCode 插件：[MASM/TASM](https://marketplace.visualstudio.com/items?itemName=xsro.masm-tasm)
这个插件还蛮方便的，省去了自己配环境的麻烦，右键单击文件选择`Run ASM code`就可以执行代码：  

![upload successful](/images/vsc_run_code.png)


DOS系统功能调用格式都是一致的，步骤如下：
1. 在`AH`寄存器中设置系统功能调用号。
2. 在指定的寄存器中设置入口参数。
3. 用`INT 21H`指令执行功能调用。
4. 根据出口参数分析功能调用执行情况。

## 常见功能

DOS 系统功能调用 `INT 21H`，有数百种功能供用户使用。下面介绍几个常用的 DOS 系统功能调用，简要描述如表所示。  

![upload successful](/images/dos_api.png)  
4C肯定天天用，毕竟用来返回DOS的，这里写点其他功能的例子。  


## 一些例子
汇编代码中，大写和小写没关系，所以这里ah和AH，dx和DX也没问题。还有4H和4也是一样的，汇编中默认都是16进制。

#### 显示一个字符：
```
assume cs:code

code segment

start:
    MOV  DL,'l'
    ;调用DOS系统功能，2表示输出DL寄存器的字符到显示器
    MOV  AH,2h
    INT  21H
    ;返回DOS
    MOV AH, 4CH
    INT 21H

code ends
end start
```


#### 打印hello world
代码是用别人的，他的注释写的蛮好的：
```
; 注：这里的关联并没有任何实际操作，相当于给我们自己的注释而已
; 相当于即使不写这一行也没有关系
assume cs:code, ds:data
  
; 数据段
data segment  
    ; 创建字符串
    ; 汇编打印字符串要在尾部用 $ 标记字符串的结束位置
    ; 将字符串用hello做一个标记，方便后面使用它
    hello db 'Hello World!$'
data ends

; 代码段
code segment  
; 指令执行的起始，类似于C语言的main函数入口
start:  
    ; 汇编语言不会自动把数据段寄存器指向我们程序的数据段
    ; 将数据段寄存器指向我们自己程序的数据段
    mov ax, data
    mov ds, ax

    ; 打印字符串的参数
    ; DS:DX=串地址，将字符串的偏移地址传入dx寄存器
    ; 字符串是在数据段起始创建的，它的偏移地址是0H
    ; offset hello 即找到标记为hello的数据段字符串的编译地址
    ; 还可以写成 mov dx, 0H
    mov dx, offset hello  
    ; 打印字符串，ah=9H代表打印
    mov AH, 9H
    int 21H
    
    ; 正常退出程序，相当于高级语言的 return 0
    mov AH, 4CH
    int 21H
     
code ends     
end start
```



参考：
* [从零入门8086汇编](https://juejin.cn/post/6844903866153041928)