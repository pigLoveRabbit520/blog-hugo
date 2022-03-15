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
汇编代码中，指令和寄存器名大写和小写没关系，所以这里ah和AH，dx和DX也没问题。但是4H和4还是有点不一样的，`4H`是16进制，不带h表示10进制。

#### 显示一个字符
```
assume cs:code, ds:data
  
; 数据段
data segment  
    message db 'hello'
data ends

; 代码段
code segment  
start: 
    mov ax, data
    mov ds, ax

    mov dl, ds:[1h]

    ;调用DOS系统功能，2表示输出DL寄存器的字符到显示器
    mov ah, 2h
    int 21h
    
    ;返回DOS
    mov ah, 4ch
    int 21h
     
code ends     
end start
```
这个程序会输出一个`e`的字符。

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
    mov ah, 9h
    int 21h
    
    ; 正常退出程序，相当于高级语言的 return 0
    mov ah, 4ch
    int 21h
     
code ends     
end start
```

#### 读取键盘输入
这个例子复杂了点
```
assume cs:code, ds:data, ss:stack

stack segment
    dw 30h dup(0)
stack ends
  
; 数据段
data segment
    buf db 20h, 0, 20h dup (0)
    message db 'input a number:$'
    num dw ?
data ends

code segment
start:
    mov ax, data
    mov ds, ax
    call printMsg
    call readInput
    call atoi
    push num

    mov dl, 0Ah
    mov ah, 02h
    int 21h

    ; again
    call printMsg
    call readInput
    call atoi

    mov ah, 4ch
    int 21h

printMsg:
    mov dx, offset message
    mov ah, 9h
    int 21h
    ret

readInput:
    ; first byte to tell dos maximum characters buffer can hold
    mov dx, 0h
    mov ah, 0Ah
    int 21h
    ret

atoi proc
	mov dx,0
	mov bx,10
	mov si,2
	mov num,0
	mov ax,0
lop:
    mov al,buf[si]
    cmp al,0Dh
    je  final
    sub al,30h
    cmp num,0
    je  do_delta
    push ax
    mov ax,num
    mul bx
    mov num,ax  
    pop ax
do_delta:
    add num,ax
    mov ax,0
    inc si
    jmp lop
final:    
    ret
atoi endp

code ends     
end start
```


## 调试工具DEBUG常用命令

##### R ——查看和修改寄存器
##### D ——查看内存单元
内存每16个字节单元为一小段，逻辑段必须从小段的首址开始。用D命令可以查看存储单元的地址和内容。  
D命令格式为：  
```
D  段地址:起始偏移地址 [结尾偏移地址] [L范围]
```
例如：  
```
D DS:0      查看数据段，从0号单元开始  
D ES:0      查看附加段，从0号单元开始  
D DS:100   查看数据段，从100H号单元开始  
D 0200:5 15   查看0200H段的5号单元到15H号单元（在虚拟机上该命令不能执行）  
D 0200:5 L 11  用L选择范围。查看0200H段的5号单元到15H号单元共10个单元  
```
##### T /P——单步执行
P可以跳过子程序或系统调用，其他方面T和P是类型的。


参考：
* [从零入门8086汇编](https://juejin.cn/post/6844903866153041928)
* [调试工具DEBUG](https://www.cnblogs.com/lfri/p/10780994.html)