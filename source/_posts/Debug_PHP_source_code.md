title: 调试PHP源码
author: Salamander
tags:
  - PHP
  - Linux
categories:
  - PHP
date: 2020-07-03 20:00:00
---
![](https://s1.ax1x.com/2020/07/05/U91xmt.png)

## 缘由
有时候，我们想看看一个变量底层对应底层的数据结构或者PHP脚本是如何执行的，gdb就是这样一个好工具，之前有篇[文章写过](/2020/07/02/gdb_use/)如何简单使用gdb。


<!-- more -->


## 编译
你可以从[PHP官网下载PHP源码的压缩包](https://www.php.net/downloads)，者是从git.php.net（或者是github的镜像）的git库clone最新的代码库，然后切换到对应的PHP版本的分支，本文使用的是PHP7.1，你可以使用下面的命令完成这些工作：
```
git clone http://git.php.net/repository/php-src.git
cd php-src
git checkout PHP-7.1
```
如果你是从git库中clone的代码，那么你先要运行下buildconf命令：
```
~/php-src> ./buildconf 
```
这个命令会生成configure脚本，**从官网下载的源码包中会直接包含这个脚本**，如果你执行`buildconf`出错，那么很可能是因为你的系统中没有`autoconf`这个工具，你可以使用包安装工具进行安装。  
如果你已经成功生成了configure脚本文件（或者是使用已包含这个脚本文件的源码包），那就可以开始编译了。为了调式PHP源码，我们的编译会disable所有的扩展（除了一些必须包含的外，这些PHP的编译脚本会自行处理），我们使用下面的命令来完成编译安装的工作，假设安装的路径为$HOME/myphp：
```
~/php-src> ./configure --disable-all --enable-debug --prefix=$HOME/myphp
~/php-src> make -jN
~/php-src> make install
```
注意这里的prefix的参数必须为绝对路径，所以你不能写成~/myphp，另外我们这次编译只是为了调式，所以建议一定要设置prefix参数，要不然PHP会被安装到默认路径中，大多数时候是/usr/local/php中，这可能会造成一些没必要的污染。另外我们使用了两个选项，一个是--disable-all，这个表示禁止安装所有扩展（除了一个必须安装的），另外一个就是--enable-debug，这个选项表示以debug模式编译PHP源码，**相当于gcc的-g选项**，它会把调试信息编译进最终的二进制程序中。  

上面的命令make -jN，N表示你的CPU数量（或者是CPU核心的数量），设置了这个参数后就可以使用多个CPU进行并行编译，这可以提高编译效率。