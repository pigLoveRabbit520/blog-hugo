---
title: C语言动态链接库回顾
author: Salamander
tags:
  - C
categories:
  - C
date: 2020-05-03 10:00:00
---
![C language](https://s1.ax1x.com/2020/05/09/YMO4bV.jpg)

## 动态链接库和静态链接库
**静态链接库**会在编译时包含到可执行文件中，这样的程序虽然没有依赖问题，但是可执行文件体积较大，包含相同的公共代码，非常浪费内存。  
动态链接库的好处就是节省内存空间，还有将一些程序升级变得简单。用户只需要升级动态链接库，而无需重新编译链接其他原有的代码就可以完成整个程序的升级。  
在windows下动态链接库是以`.dll`后缀的文件，静态链接库是以`.lib`的文件  
而在Linux中，动态链接库是以`.so`作后缀的文件，静态链接库是以`.a`（archive的缩写）的文件。  
本文中，我们的链接库来自于[ftplib](https://nbpfaus.net/~pfau/ftplib/)，这是一个用C语言实现的跨平台FTP库，我们将会用它生成的动态链接库写个简单的程序（连接ftp服务器，然后查询当前目录）。


<!-- more -->

本文环境：
* OS：Ubuntu 18.04.4 LTS 还有 Windows 10专业版
* ftplib：V4.0-1
* gcc： 7.5.0 (Ubuntu 7.5.0-3ubuntu1~18.04)


## Linux下
在Linux中，标准库的大部分函数通常放在文件 libc.a 中（这是**静态链接库**），或者放在用于共享的动态链接文件 libc.so 中（文件名后缀.so代表“share object”，译为“共享对象”，这是**动态链接库**）。这些链接库一般位于 /lib/ 或 /usr/lib/，或者位于 GCC 默认搜索的其他目录（默认搜索目录有`/lib` `/usr/lib` `/usr/local/lib`）。  
观察一下**ftplib**的`Makefile`
```makefile
....

install : all
	install qftp /usr/local/bin
	install -m 644 libftp.so.$(SOVERSION) /usr/local/lib
	install -m 644 ftplib.h /usr/local/include
	(cd /usr/local/lib && \
	 ln -sf libftp.so.$(SOVERSION) libftp.so.$(SONAME) && \
	 ln -sf libftp.so.$(SONAME) libftp.so)
	-(cd /usr/local/bin && \
	  for f in ftpdir ftpget ftplist ftprm ftpsend; \
	  do ln -s qftp $$f; done)
...
```
`make install`的时候会把`ftplib.h`放到`/usr/local/include`，把`libftp.so`放到`/usr/local/lib`，一个是gcc默认的头文件搜索目录，一个是gcc默认的库文件搜索目录。 

## 安装动态链接库

在ftplib的src目录中执行
```shell
make
sudo make install
```
以上操作会把动态链接库放入到我们的系统中。  


### 程序引用
然后我们写一个简单的程序
```C
#include <stdio.h>
#include "ftplib.h"

netbuf *con = NULL;
char host[] = "192.168.1.175:2121"; // 小米手机的ftp服务
char username[] = "**********";
char password[] = "********";

int main()
{
    if(!FtpConnect(host, &con))
    {
        printf("connect failed!!\n");
        return 0;
    }

    // 登录
    if(!FtpLogin(username, password, con))
    {
        printf("login failed!\n");
        FtpQuit(con);
        return 0;
    }
    printf("Login successfully!\n");
    char currentDir[20];
    FtpPwd(currentDir, 10, con);
    printf("Current Directory is %s\n", currentDir);
    FtpQuit(con);
    return 0;
}
```
好了，现在我们可以用gcc编译它了，因为这里引用了`ftplib`库，所以我们需要手动添加链接库
```
gcc main.c -o main /usr/local/lib/libftp.so
```
上面使用了链接库的完整路径，其实我们可以用`-l`选项，因为生成的链接库命名是规范的（`-lXX`会去找`libxx.so`这样的文件，自动加`lib`前缀）而且也在gcc的默认搜索目录中
```
gcc main.c -o main -lftp
```
另外说一句，有时候我们想增加一个自定义的搜索目录，可以使用`-L`选项，例如
```
gcc main.c -o main.out -L/usr/lib -lhello
```
另外也可以使用环境变量`LD_LIBRARY_PATH`指定搜索目录（在程序执行之前定义这个量就行），路径之间用冒号”:”分隔。  
我们可以用`ldd`命令查看程序用到的链接库：
```
$ ldd main
        linux-vdso.so.1 (0x00007fff7a7fa000)
        libftp.so.4 => /usr/local/lib/libftp.so.4 (0x00007f7b5215a000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f7b51d69000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f7b52563000)
```








参考：
* [Shared libraries with GCC on Linux](https://www.cprogramming.com/tutorial/shared-libraries-linux-gcc.html)