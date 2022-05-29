title: bochs下启动自己的MBR
author: pigLoveRabbit
tags:
  - OS
categories:
  - OS
date: 2022-04-17 23:12:00
---
# Bochs
Bochs是一个x86硬件平台的开源模拟器。你可以当它是一台虚拟的x86的计算机。


<!-- more -->


本文环境：  
* OS: Ubuntu 20.04.4 LTS
* Bochs： 2.6.9


## 安装Bochs
这个网上教程很多了，先去[下载](https://sourceforge.net/projects/bochs/files/bochs/)，然后configure：
```
./configure \
--prefix=/your_path/bochs \
--enable-debugger\
--enable-disasm \
--enable-iodebug \
--enable-x86-debugger \
--with-x \
--with-x11
```
这里会出点问题，一般是缺少库了：
```
sudo apt-get install libx11-dev ................. for X11/Xlib.h
sudo apt-get install mesa-common-dev........ for GL/glx.h
sudo apt-get install libglu1-mesa-dev ..... for GL/glu.h
sudo apt-get install libxrandr-dev ........... for X11/extensions/Xrandr.h
sudo apt-get install libxi-dev ................... for X11/extensions/XInput.h
```
configure通过就可以编译安装了：
```
make
sudo make install
```

## 配置
当我们在终端输入bochs后，
Bochs会自己在当前目录顺序寻找以下文件作为默认配置文件：
.bochsrc  
bochsrc  
bochsrc.txt  
bochsrc.bxrc(仅对Windows有效)  
我们可以自己创建一个名为.bochsrc的文件，来指定Bochs配置我们想要的虚拟机。  
```
[~/bochs_fun]$ cat bochsrc

#################################################################
# Bochs的配置文件
# Configuration file for Bochs
#################################################################

# how much memory the emulated machine will have
megs: 32

# filenameof ROM images
romimage:file=/usr/local/share/bochs/BIOS-bochs-latest
vgaromimage:file=/usr/local/share/bochs/VGABIOS-lgpl-latest

# which disk image will be used 这个是启动软盘
floppya:1_44=a.img, status=inserted

# choose the boot disk 确定启动方式
#boot: floppy
boot: disk

# where do we send log messages?
log: bochsout.txt

# disable the mouse
mouse: enabled=0

# enable key mapping ,using US layout as default
keyboard:keymap=/usr/local/share/bochs/keymaps/x11-pc-us.map

# 硬盘设置
ata0: enabled=1, ioaddr1=0x1f0, ioaddr2=0x3f0, irq=14
```

## 测试开机

按`c`继续运行，弹出新的窗口。  





## MBR
MBR：Master Boot Record，主分区引导记录，它是整个硬盘最开始的扇区，即0柱面0磁头1扇区（CHS表示方法，如果是LBA的话，那就是0扇区）。  

扇区是什么？那就涉及到硬盘基本知识了，对于机械硬盘，有以下概念：
* 盘片（platter）
* 磁头（head）
* 磁道（track）
* 扇区（sector）
* 柱面（cylinder）

**盘片 片面 和 磁头（Head）**

硬盘中一般会有多个盘片组成，每个盘片包含两个面，每个盘面都对应地有一个**读/写磁头**。受到硬盘整体体积和生产成本的限制，盘片数量都受到限制，一般都在5片以内。盘片的编号自下向上从0开始，如最下边的盘片有0面和1面，再上一个盘片就编号为2面和3面。  


![upload successful](/images/disk_image.png)  

**扇区（Sector）和 磁道（Track）**  

下图显示的是一个盘面，盘面中一圈圈灰色同心圆为一条条磁道，从圆心向外画直线，可以将磁道划分为若干个弧段，每个磁道上一个弧段被称之为一个扇区（图践绿色部分）。扇区是磁盘的最小组成单元，通常是512字节。（由于不断提高磁盘的大小，部分厂商设定每个扇区的大小是4096字节）  


![upload successful](/images/disk_sector.png)



**磁头（Head）和 柱面（Cylinder）**
硬盘通常由重叠的一组盘片构成，每个盘面都被划分为数目相等的磁道，并从外缘的“0”开始编号，具有相同编号的磁道形成一个圆柱，称之为磁盘的柱面。磁盘的柱面数与一个盘面上的磁道数是相等的。由于每个盘面都有自己的磁头，因此，盘面数等于总的磁头数。 如下图  

![upload successful](/images/disk_cylinder.png)  图3




所谓CHS即柱面（cylinder），磁头（header），扇区（sector），通过这三个变量描述磁盘地址（先定位柱面，然后选定磁头，然后确定扇区），需要明白的是，这里表示的已不是物理地址而是逻辑地址了。这种方法也称作是LARGE寻址方式。该方法下：  

硬盘容量=磁头数×柱面数×扇区数×扇区大小（一般为512byte）。  

后来，人们通过为每个扇区分配逻辑地址，以扇区为单位进行寻址，也就有了LBA寻址方式。但是为了保持与CHS模式的兼容，通过逻辑变换算法，可以转换为磁头/柱面/扇区三种参数来表示，和 LARGE寻址模式一样，这里的地址也是逻辑地址了。（固态硬盘的存储原理虽然与机械硬盘不同，采用的是**flash**存储，但仍然使用LBA进行管理，此处不再详述。）  


好了，再回来说MBR，MBR(Master Boot Record)是传统的分区机制，应用于绝大多数使用BIOS引导的PC设备（苹果使用EFI的方式），很多Server服务器即支持BIOS也支持EFI的引导方式。它的结构如下：  
MBR结构：占用硬盘最开头的512字节

 * 前446字节为：引导代码（Bootstrap Code Area）（引导不同的操作系统；不同操作系统，引导代码是不一样的）  
 * 接下来的为4个16字节：分别对应4个主分区表信息(Primary Partition Table)
 * 最后2个字节：为启动标示（Boot Signature），永远都是`55`和`AA`；55和ＡＡ是个永久性的标示，代表这个硬盘是可启动的。
 
 
![upload successful](/images/mbr_structure.png)  



## 主引导代码
BIOS在完成一些简单的检测工作或初始化工作后，会把处理器使用权交出去，下一棒就是MBR程序。BIOS会把MBR加载到0x7c00的位置，然后执行里头代码（`jmp 0:0x7c00`）。  
**mbr.S**文件：
```
;主引导程序 
;------------------------------------------------------------
SECTION MBR vstart=0x7c00
   mov ax,cs      
   mov ds,ax
   mov es,ax
   mov ss,ax
   mov fs,ax
   mov sp,0x7c00

; 清屏 利用0x06号功能，上卷全部行，则可清屏。
; -----------------------------------------------------------
;INT 0x10   功能号:0x06	   功能描述:上卷窗口
;------------------------------------------------------
;输入：
;AH 功能号= 0x06
;AL = 上卷的行数(如果为0,表示全部)
;BH = 上卷行属性
;(CL,CH) = 窗口左上角的(X,Y)位置
;(DL,DH) = 窗口右下角的(X,Y)位置
;无返回值：
   mov     ax, 0x600
   mov     bx, 0x700
   mov     cx, 0           ; 左上角: (0, 0)
   mov     dx, 0x184f	   ; 右下角: (80,25),
			   ; VGA文本模式中,一行只能容纳80个字符,共25行。
			   ; 下标从0开始,所以0x18=24,0x4f=79
   int     0x10            ; int 0x10

;;;;;;;;;    下面这三行代码是获取光标位置    ;;;;;;;;;
;.get_cursor获取当前光标位置,在光标位置处打印字符.
   mov ah, 3		; 输入: 3号子功能是获取光标位置,需要存入ah寄存器
   mov bh, 0		; bh寄存器存储的是待获取光标的页号

   int 0x10		; 输出: ch=光标开始行,cl=光标结束行
			; dh=光标所在行号,dl=光标所在列号

;;;;;;;;;    获取光标位置结束    ;;;;;;;;;;;;;;;;

;;;;;;;;;     打印字符串    ;;;;;;;;;;;
   ;还是用10h中断,不过这次是调用13号子功能打印字符串
   mov ax, message 
   mov bp, ax		; es:bp 为串首地址, es此时同cs一致，
			; 开头时已经为sreg初始化

   ; 光标位置要用到dx寄存器中内容,cx中的光标位置可忽略
   mov cx, 0xb		; cx 为串长度,不包括结束符0的字符个数
   mov ax, 0x1301	; 子功能号13是显示字符及属性,要存入ah寄存器,
			; al设置写字符方式 ah=01: 显示字符串,光标跟随移动
   mov bx, 0x2		; bh存储要显示的页号,此处是第0页,
			; bl中是字符属性, 属性黑底绿字(bl = 02h)
   int 0x10		; 执行BIOS 0x10 号中断
;;;;;;;;;      打字字符串结束	 ;;;;;;;;;;;;;;;

   jmp $		; 使程序悬停在此

   message db "love rabbit"
   times 510-($-$$) db 0
   db 0x55,0xaa
```
用nasm编译成纯二进制文件`nasm -o mbr.bin mbr.S`，可以查看**mbr.bin**文件大小，正好512个字节。  

然后利用`dd`命令把bin文件写进磁盘的0柱面0磁头1扇区：
```
dd if=/your_path/bochs_fun/mbr.bin of=/your_path/bochs_fun/hd60M.img bs=512 count=1 conv=notrunc
```


## 增加磁盘读写
mbr.S
```
;主引导程序 
;------------------------------------------------------------
%include "boot.inc"
SECTION MBR vstart=0x7c00         
   mov ax,cs      
   mov ds,ax
   mov es,ax
   mov ss,ax
   mov fs,ax
   mov sp,0x7c00
   mov ax,0xb800
   mov gs,ax

; 清屏
;利用0x06号功能，上卷全部行，则可清屏。
; -----------------------------------------------------------
;INT 0x10   功能号:0x06	   功能描述:上卷窗口
;------------------------------------------------------
;输入：
;AH 功能号= 0x06
;AL = 上卷的行数(如果为0,表示全部)
;BH = 上卷行属性
;(CL,CH) = 窗口左上角的(X,Y)位置
;(DL,DH) = 窗口右下角的(X,Y)位置
;无返回值：
   mov     ax, 0600h
   mov     bx, 0700h
   mov     cx, 0                   ; 左上角: (0, 0)
   mov     dx, 184fh		   ; 右下角: (80,25),
				   ; 因为VGA文本模式中，一行只能容纳80个字符,共25行。
				   ; 下标从0开始，所以0x18=24,0x4f=79
   int     10h                     ; int 10h

   ; 输出字符串:MBR
   mov byte [gs:0x00],'1'
   mov byte [gs:0x01],0xA4

   mov byte [gs:0x02],' '
   mov byte [gs:0x03],0xA4

   mov byte [gs:0x04],'M'
   mov byte [gs:0x05],0xA4	   ;A表示绿色背景闪烁，4表示前景色为红色

   mov byte [gs:0x06],'B'
   mov byte [gs:0x07],0xA4

   mov byte [gs:0x08],'R'
   mov byte [gs:0x09],0xA4
	 
   mov eax,LOADER_START_SECTOR	 ; 起始扇区lba地址
   mov bx,LOADER_BASE_ADDR       ; 写入的地址
   mov cx,1			 ; 待读入的扇区数
   call rd_disk_m_16		 ; 以下读取程序的起始部分（一个扇区）
  
   jmp LOADER_BASE_ADDR
       
;-------------------------------------------------------------------------------
;功能:读取硬盘n个扇区
rd_disk_m_16:	   
;-------------------------------------------------------------------------------
				       ; eax=LBA扇区号
				       ; ebx=将数据写入的内存地址
				       ; ecx=读入的扇区数
      mov esi,eax	  ;备份eax
      mov di,cx		  ;备份cx
;读写硬盘:
;第1步：设置要读取的扇区数
      mov dx,0x1f2
      mov al,cl
      out dx,al            ;读取的扇区数

      mov eax,esi	   ;恢复ax

;第2步：将LBA地址存入0x1f3 ~ 0x1f6

      ;LBA地址7~0位写入端口0x1f3
      mov dx,0x1f3                       
      out dx,al                          

      ;LBA地址15~8位写入端口0x1f4
      mov cl,8
      shr eax,cl
      mov dx,0x1f4
      out dx,al

      ;LBA地址23~16位写入端口0x1f5
      shr eax,cl
      mov dx,0x1f5
      out dx,al

      shr eax,cl
      and al,0x0f	   ;lba第24~27位
      or al,0xe0	   ; 设置7～4位为1110,表示lba模式
      mov dx,0x1f6
      out dx,al

;第3步：向0x1f7端口写入读命令，0x20 
      mov dx,0x1f7
      mov al,0x20                        
      out dx,al

;第4步：检测硬盘状态
  .not_ready:
      ;同一端口，写时表示写入命令字，读时表示读入硬盘状态
      nop
      in al,dx
      and al,0x88	   ;第4位为1表示硬盘控制器已准备好数据传输，第7位为1表示硬盘忙
      cmp al,0x08
      jnz .not_ready	   ;若未准备好，继续等。

;第5步：从0x1f0端口读数据
      mov ax, di
      mov dx, 256
      mul dx
      mov cx, ax	   ; di为要读取的扇区数，一个扇区有512字节，每次读入一个字，
			   ; 共需di*512/2次，所以di*256
      mov dx, 0x1f0
  .go_on_read:
      in ax,dx
      mov [bx],ax
      add bx,2		  
      loop .go_on_read
      ret

   times 510-($-$$) db 0
   db 0x55,0xaa
```
**boot.inc**文件
```
;-------------	 loader和kernel   ----------
LOADER_BASE_ADDR equ 0x900 
LOADER_START_SECTOR equ 0x2
```
最后CPU会调到0x900的地址，还得有个loader
```
%include "boot.inc"
section loader vstart=LOADER_BASE_ADDR

; 输出背景色绿色，前景色红色，并且跳动的字符串"1 MBR"
mov byte [gs:0x20],'2'
mov byte [gs:0x21],0xA4     ; A表示绿色背景闪烁，4表示前景色为红色

mov byte [gs:0x22],' '
mov byte [gs:0x23],0xA4

mov byte [gs:0x24],'L'
mov byte [gs:0x25],0xA4   

mov byte [gs:0x26],'O'
mov byte [gs:0x27],0xA4

mov byte [gs:0x28],'A'
mov byte [gs:0x29],0xA4

mov byte [gs:0x2a],'D'
mov byte [gs:0x2b],0xA4

mov byte [gs:0x2c],'E'
mov byte [gs:0x2d],0xA4

mov byte [gs:0x2e],'R'
mov byte [gs:0x2f],0xA4

jmp $		       ; 通过死循环使程序悬停在此
```


参考：
* [OS篇-Bochs在Ubuntu下的安装教程](https://mikeygithub.github.io/2021/02/26/os/OS%E7%AF%87-Bochs%E5%9C%A8Ubuntu%E4%B8%8B%E7%9A%84%E5%AE%89%E8%A3%85%E6%95%99%E7%A8%8B/)
* [Ubuntu20.04/18.04安装Bochs2.6.9编译运行GeekOS](https://www.cxybb.com/article/Java_c110/115737121)
* [柱面-磁头-扇区寻址的一些旧事 ](https://farseerfc.me/zhs/history-of-chs-addressing.html)
* [MBR与GPT](https://zhuanlan.zhihu.com/p/26098509)