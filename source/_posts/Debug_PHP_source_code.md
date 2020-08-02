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

本文环境：
* PHP版本：PHP 7.1.16 (cli) (built: Apr  8 2020 11:56:59) ( ZTS )
* OS：Ubuntu 18.04.4 LTS
* gdb: GNU gdb (Ubuntu 8.1-0ubuntu3.2) 8.1.0.20180409-git


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


## 调试PHP
我们调试一段简单的PHP代码：
```
<?php
$a = 10;
$b = 42;

echo $b;
```
我们想看下`$a`对应的底层变量结构，那我们应该在哪个函数上叫断点呢？通过查阅资料（如《PHP7内核分析》）我们发现，ZendVM的执行器就是一个white循环，在这个循环中依次调用`opline`指令的handler，然后根据handler的返回决定下一步的动作。执行调度器为**zend_execute_ex**，这是函数指针，默认为`execute_ex`，我们看下这个函数的代码：
```
//删除了预处理语句
ZEND_API void execute_ex(zend_execute_data *ex)
{
    DCL_OPLINE

    const zend_op *orig_opline = opline;
    zend_execute_data *orig_execute_data = execute_data; /* execute_data是一个全局变量 */
    execute_data = ex; 


    LOAD_OPLINE();

    while (1) {
        ((opcode_handler_t)OPLINE->handler)(ZEND_OPCODE_HANDLER_ARGS_PASSTHRU); //执行OPCode对应的C函数，OPLINE是一个全局变量
        if (UNEXPECTED(!OPLINE)) { //当前OPArray执行完
            execute_data = orig_execute_data;
            opline = orig_opline;
            return;
        }
    }
    zend_error_noreturn(E_CORE_ERROR, "Arrived at end of main loop which shouldn't happen");
}
```
所以我们可以在给`execute_ex`函数打断点。  
```
gdb ~/myphp/bin/php

(gdb) r index.php
Starting program: /home/salamander/myphp/bin/php index.php

Breakpoint 1, execute_ex (ex=0x7ffff7014030) at /home/salamander/php-7.1.16/Zend/zend_vm_execute.h:411
411             const zend_op *orig_opline = opline;
(gdb) n
414             zend_execute_data *orig_execute_data = execute_data;
(gdb) n
415             execute_data = ex;
(gdb) n
421             LOAD_OPLINE();
(gdb) n
422             ZEND_VM_LOOP_INTERRUPT_CHECK();
(gdb) n
429                     ((opcode_handler_t)OPLINE->handler)(ZEND_OPCODE_HANDLER_ARGS_PASSTHRU);
```
现在就要调用`opline`指令的handler，我们应该键入`s`，跳到对应函数内部去： 
```
(gdb) s
ZEND_ASSIGN_SPEC_CV_CONST_RETVAL_UNUSED_HANDLER () at /home/salamander/php-7.1.16/Zend/zend_vm_execute.h:39506
39506           SAVE_OPLINE();
(gdb) n
39507           value = EX_CONSTANT(opline->op2);
(gdb) n
39508           variable_ptr = _get_zval_ptr_cv_undef_BP_VAR_W(execute_data, opline->op1.var);
(gdb) n
39516                   value = zend_assign_to_variable(variable_ptr, value, IS_CONST);
(gdb) p value
$1 = (zval *) 0x7ffff707b460
(gdb) p *$1
$3 = {value = {lval = 10, dval = 4.9406564584124654e-323, counted = 0xa, str = 0xa, arr = 0xa, obj = 0xa, res = 0xa, ref = 0xa, ast = 0xa, zv = 0xa, 
    ptr = 0xa, ce = 0xa, func = 0xa, ww = {w1 = 10, w2 = 0}}, u1 = {v = {type = 4 '\004', type_flags = 0 '\000', const_flags = 0 '\000', 
      reserved = 0 '\000'}, type_info = 4}, u2 = {next = 4294967295, cache_slot = 4294967295, lineno = 4294967295, num_args = 4294967295, 
    fe_pos = 4294967295, fe_iter_idx = 4294967295, access_flags = 4294967295, property_guard = 4294967295, extra = 4294967295}}
```
我们第一行PHP代码是`$a = 10;`，这是一条赋值语句,`ZEND_ASSIGN_SPEC_CV_CONST_RETVAL_UNUSED_HANDLER`是把一个常量赋值给一个变量，`EX_CONSTANT(opline->op2)`是获取常量的值，`$a`为CV变量，分配在zend_execute_data动态变量区，通过`_get_zval_ptr_cv_undef_BP_VAR_W`取到这个变量的地址，剩下的好理解了，就是把变量值赋值给CV变量。  
`value`就是我们的变量值，`$a`对应的底层变量就是它。




参考：
* [PHP 7 中函数调用的实现](http://yangxikun.github.io/php/2016/11/04/php-7-func-call.html)