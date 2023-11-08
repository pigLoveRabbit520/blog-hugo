---
title: gdb调试简单使用
author: Salamander
tags:
  - gdb
  - Linux
categories:
  - Linux
date: 2020-07-02 11:00:00
---
## gdb
gdb 是 UNIX 及 UNIX-like 下的调试工具，在 Linux 下一般都直接在命令行中用 gdb 来调试程序，相比 Windows 上的集成开发环境 IDE 提供的图形界面调试，一开始使用 gdb 调试可能会让你感觉很难适应，但是只要熟悉了 gdb 调试的常用命令，调试出程序会很有成就感，一方面因为这些命令就类似图形界面调试按钮背后的逻辑，另一方面用命令行来调试程序，逼格瞬间就上了一个档次，这次就跟大家分享 gdb 调试的基本技术和 15 个常用调试命令。

<!-- more -->

## 使用

### gdb快捷键说明
 ```
 一些快捷命令

l – list
p – print print {variable}  //打印变量
c – continue           //继续执行
s – step          
b - break break line_number/break [file_name]:line_number/break [file_name]:func_name       //设置断点
r - run                    //执行文件
```

### 使用
#### 编译可以调试的程序
这是本次要调试的 hello.c 程序，非常简单：
```
#include <stdio.h>

int add(int x, int y) {
	return x + y;
}

int main() {
	int a = 1;
	int b = 2;
	printf("a = %d\n", a);
	printf("b = %d\n", b);

	int c = add(a, b);
	printf("%d + %d = %d\n", a, b, c);
	return 0;
}
```
我们平常使用 gcc 编译的程序如果不加 [-g] 选项：
```
gcc hello.c -o hello
```
gdb 会提示该可执行文件没有调试符号，不能调试：
```
gdb hello
...
Reading symbols from hello...(no debugging symbols found)...done.
...
```
如果需要让程序可以调试，就**必须在编译的时候加上 ** `[-g]` 参数

#### 载入要调试的程序
使用如下的命令来载入可执行文件 hello 到 gdb 中：
```
gdb hello
```
载入成功，gdb 会打印一段提示信息，并且命令行前缀变为 (gdb)，下面是我的 Ubuntu 输出的信息：
```
GNU gdb (Ubuntu 8.1-0ubuntu3.2) 8.1.0.20180409-git
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from hello...done.
(gdb) 
```
注，**按 q 退出 gdb**  

方法二 - 使用 gdb 提供的 file 命令  
第二种方法是在 gdb 环境中使用 file 命令，我们需要先进入 gdb 环境下：
```
gdb
```
使用 file hello 载入待调试程序：
```
...
(gdb) file hello
Reading symbols from hello...done.
(gdb) q
```

#### 查看调试程序
在 gdb 下查看调试程序使用命令 `list` 或简写 `l`，「回车」列出后面程序：
```
(gdb) list
1       #include <stdio.h>
2
3       int add(int x, int y) {
4               return x + y;
5       }
6
7       int main() {
8               int a = 1;
9               int b = 2;
10              printf("a = %d\n", a);
(gdb) 
```

#### 添加断点
在 gdb 下添加断点使用命令 `break` 或简写 `b`，有下面几个常见用法（这里统一用 `b`）：
1. b function_name
2. b row_num
3. b file_name:row_num
4. b row_num if condition

比如我们以第一个为例，在 `main` 函数上添加断点：
```
(gdb) b main
Breakpoint 1 at 0x666: file hello.c, line 8.
```
打印的信息告诉我们在 hello.c 文件的第 8 行，地址 0x666 处添加了一个断点，那如何查看断点呢？  
#### 查看断点
在 gdb 下查看断点使用命令 `info break` 或简写 `i b`，比如查看刚才打的断点：
```
(gdb) i b
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x0000000000000666 in main at hello.c:8
```

#### 删除断点
在 gdb 下删除断点使用命令 delete 断点 Num 或简写 d Num，比如删除刚才的 Num = 1 的断点：
```
(gdb) d 1
(gdb) i b
No breakpoints or watchpoints.
```
删除后再次查看断点，提示当前没有断点，说明删除成功啦，下面来运行程序试试。


#### 运行程序
在 gdb 下使用命令 run 或简写 r 来运行当前载入的程序：
```
(gdb) r
Starting program: /home/salamander/文档/test/hello 
a = 1
b = 2
1 + 2 = 3
[Inferior 1 (process 16249) exited normally]
```
我这次没有添加断点，程序全速运行，然后正常退出了。

#### 单步执行下一步
在 gdb 下使用命令 `next` 或简写 `n` 来单步执行下一步，假设我们在 `main` 打了断点：
```
(gdb) b main
Breakpoint 1 at 0x555555554666: file hello.c, line 8.
(gdb) r
Starting program: /home/salamander/文档/test/hello 

Breakpoint 1, main () at hello.c:8
8               int a = 1;
(gdb) n
9               int b = 2;
```
可以看到当前停在 int a = 1; 这一行，按 n 执行了下一句代码 `int b = 2;`


#### 打印变量
在 gdb 中使用命令 print var 或简写 p var 来打印一个变量或者函数的返回值，在上述gdb中打印 a 的值：
```
(gdb) b main
Breakpoint 1 at 0x555555554666: file hello.c, line 8.
(gdb) r
Starting program: /home/salamander/文档/test/hello 

Breakpoint 1, main () at hello.c:8
8               int a = 1;
(gdb) n
9               int b = 2;
(gdb) n
10              printf("a = %d\n", a);
(gdb) p a
$1 = 1
```



参考：
* [Linux 高级编程 - 15 个 gdb 调试基础命令](https://dlonng.com/posts/gdb)