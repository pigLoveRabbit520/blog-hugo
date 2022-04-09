title: Win32汇编
author: pigLoveRabbit
date: 2022-04-08 21:09:38
tags:
---
## Win32汇编
8086的指令在32位的x86 CPU上都是能用的，所以汇编代码是类似的，Win32 **保护模式**下段寄存器就不重要了，平常用的多的就是那8个通用寄存器（为啥用不到CS, DS, ES等寄存器了，请看[这里](https://en.wikipedia.org/wiki/X86_memory_segmentation)）：   

| 通用寄存器  | 含义 | 低8位 | 功能 |
| --- | --- | --- | --- |
| EAX     | 累加(Accumulator)寄存器                                                                                       | AX(AH、AL)                     | 常用于乘、除法和函数返回值                                                     |
| EBX     | 基址(Base)寄存器                                                                                              | BX(BH、BL)                     | 常做内存数据的指针, 或者说常以它为基址来访问内存.                                        |
| ECX     | 计数器(Counter)寄存器                                                                                          | CX(CH、CL)                     | 常做字符串和循环操作中的计数器                                                   |
| EDX     | 数据(Data)寄存器                                                                                              | DX(DH、DL)                     | 常用于乘、除法和 I/O 指针                                                   |
| ESI     | 来源索引(Source Index)寄存器                                                                                    | SI                            | 常做内存数据指针和源字符串指针                                                   |
| EDI     | 目的索引(Destination Index)寄存器                                                                               | DI                            | 常做内存数据指针和目的字符串指针                                                  |
| ESP     | 堆栈指针(Stack Point)寄存器                                                                                     | SP                            | 只做堆栈的栈顶指针; 不能用于算术运算与数据传送                                          |
| EBP     | 基址指针(Base Point)寄存器                                                                                      | BP                            | 只做堆栈指针, 可以访问堆栈内任意地址, 经常用于中转 ESP 中的数据, 也常以它为基址来访问堆栈; 不能用于算术运算与数据传送 |




<!-- more -->



## 调用MessageBox
```Assembly
; Message Box, 32 bit. V1.01
NULL          EQU 0                             ; Constants
MB_DEFBUTTON1 EQU 0
MB_DEFBUTTON2 EQU 100h
IDNO          EQU 7
MB_YESNO      EQU 4

extern _MessageBoxA@16                          ; Import external symbols
extern _ExitProcess@4                           ; Windows API functions, decorated

global Start                                    ; Export symbols. The entry point

section .data                                   ; Initialized data segment
 MessageBoxText    db "Do you want to exit?", 0
 MessageBoxCaption db "MessageBox 32", 0

section .text                                   ; Code segment
Start:
 push  MB_YESNO | MB_DEFBUTTON2                 ; 4th parameter. 2 constants ORed together
 push  MessageBoxCaption                        ; 3rd parameter
 push  MessageBoxText                           ; 2nd parameter
 push  NULL                                     ; 1st parameter
 call  _MessageBoxA@16

 cmp   EAX, IDNO                                ; Check the return value for "No"
 je    Start

 push  NULL
 call  _ExitProcess@4

```
用nasm编译`nasm -f win32 fun.asm -o fun.obj`  
用golink链接`golink /entry:Start kernel32.dll user32.dll fun.obj`





参考：
* [nasm-messagebox32](https://www.davidgrantham.com/nasm-messagebox32/)