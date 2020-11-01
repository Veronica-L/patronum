---
title: "Linux内核源码学习——bootsect.s"
subtitle: "Linux内核源码学习"
layout: post
author: "Veronica"
header-style: text
tags:
  - Linux
---

##### 1.启动BIOS

加电瞬间强行将CS值设置为0xF000，IP值置为0xFFF0. CS:IP=0xFFFF0,指向BIOS的地址范围

BIOS在内存最开始的位置（0x00000）用1KB构建了中断向量表，后面紧接着256字节构建BIOS数据区，大约57KB以后的位置加载了中断服务程序

`中断向量表：256个中断向量，每个中断4个字节，2个是CS，2个是IP，每个向量指向一个中断服务程序`



##### 2.把软盘中的操作系统程序加载到内存

###### 2.1 BIOS中断0x19把第一扇区bootsect内容加载进去

CPU接收到一个int 0x19中断，然后再去中断向量表里找这个中断向量。中断向量指向0x0e6f2（中断服务程序入口地址），这个中断服务程序的作用是把软盘第一扇区中的程序(512B)加载到内存中的指定位置，即将软驱0号磁头对应盘面的0磁道1扇区的内容复制到内存0x7C00处。

###### 2.2 把第二批和第三批程序加载到内存

规划内存

```assembly
.globl begtext, begdata, begbss, endtext, enddata, endbss
.text
begtext:
.data
begdata:
.bss
begbss:
.text

!对后续操作涉及的内存位置进行设置
SETUPLEN = 4					! nr of setup-sectors
BOOTSEG  = 0x07c0			! original address of boot-sector
INITSEG  = 0x9000			! we move boot here - out of the way
SETUPSEG = 0x9020			! setup starts here
SYSSEG   = 0x1000			! system loaded at 0x10000 (65536).
ENDSEG   = SYSSEG + SYSSIZE		! where to stop loading
```

复制bootsect

将源地址0x07C00的bootsect复制到目的地址0x90000

```assembly
！将源地址0x07C00的bootsect复制到目的地址0x90000
! 代码主要设定源地址和目标地址后，使用循环指令rep和移动指令movw进行移动
entry start
start:
	mov	ax,#BOOTSEG      ！BOOTSEG  = 0x07c0
	mov	ds,ax            ! ds寄存器置为0x07c0
	mov	ax,#INITSEG      ！INITSEG  = 0x9000
	mov	es,ax
	mov	cx,#256          ！计数器，提供需要复制的字的数量 256字=512字节
	sub	si,si            !sub是做减法操作，此处将si，di自己减自己，即置0。
	sub	di,di
	                !移动时源地址ds:si=0x07c0：0x0000，目的地址es:di=0x9000：0x0000
                  !即将BIOS移动到0x9000（0x07c0用于放置bootsect.s）
    rep                   !rep指令作用：重复执行后面一句操作，并递减cx的值，直到cx=0停止
    movw                  !movw指令作用：这里从内存[si]处移动cx个字到[di]；注意一次的移动单位是“字”，  mov指令+w（word）是一次移动一个字
	jmpi	go,INITSEG    !将BIOS移动到0x9000后，跳转（go）到INITSEG（0x9000），CS=0x90000
```

```assembly
! 对ds,ex,ss,sp进行调整
go:	mov	ax,cs
	mov	ds,ax
	mov	es,ax
! put stack at 0x9ff00.  !下面两条指令是将堆栈指针sp指向0x9ff00处（即0x9000:0xff00）
	mov	ss,ax
	mov	sp,#0xFF00		! arbitrary value >>512
```

将setup程序加载到内存

借助0x13中断向量，从第二个扇区开始的4个扇区

```assembly
!INT 0x13的使用方法：
!ah = 0x20-读磁盘扇区到内存； al = 需要读出的扇区数量；
!ch=磁道（柱面）号的低8位；  cl =开始扇区（位0-5），磁道号高2位（位6-7）；
!dh = 磁头号；  dl = 驱动器号；
!es：bx = 指向数据缓冲区； 
!如果出错则CF标志置位，ah中是出错码。

load_setup:
	mov	dx,#0x0000		! drive 0, head 0 磁头号0，驱动器号0
	mov	cx,#0x0002		! sector 2, track 0，开始扇区2，磁道号0
	mov	bx,#0x0200		! address = 512, in INITSEG  es:bx=0x9000:0x0200 即数据缓冲区为0x90200
	mov	ax,#0x0200+SETUPLEN	! service 2, nr of sectors,SETUPLEN初始设置为4,ax=0x0210,ah=0x02-读磁盘扇区到内存，需要读出的扇区数量-4
	int	0x13			! read it 打开中断
	jnc	ok_load_setup		! ok - continue jnc指令：如果（上条指令）成功，则跳转，即中断INT 0x13成功，则继续执行ok_load_setup
	mov	dx,#0x0000      ! 如果不成功，则复位驱动器，并重试（重新跳转函数load_setup）
	mov	ax,#0x0000		! reset the diskette
	int	0x13
	j	load_setup
```

```assembly
ok_load_setup:

! Get disk drive parameters, specifically nr of sectors/track 利用BIOS中断0x13取磁盘参数表中当前启动引导盘的参数
! 这段代码主要还是获得每磁道的扇区数量，保存在了sectors中

	mov	dl,#0x00        !驱动器号为0
	mov	ax,#0x0800		! AH=8 is get drive parameters,AH=8，是INT 0x13取磁盘驱动器的参数，AL初始化0，作为返回值
	int	0x13            ! 打开中断
	mov	ch,#0x00
	seg cs              ! 此条指令表示下一条语句的操作数在cs段寄存器所指的段中
	mov	sectors,cx
	mov	ax,#INITSEG
	mov	es,ax
```

将system模块加载到内存，加载240个扇区

```assembly
! Print some inane message
! 显示信息：“Loading system ... 回车”，共显示24个字符

! 使用BIOS中断0x10功能号ah=0x03和ah=0x13实现

! BIOS中断0x10功能号ah=0x03，功能：读光标位置
! 输入：bh=页号
! 返回：ch=扫描开始线；cl=扫描结束线；dh=行号； dl=列号

! BIOS中断0x10功能号ah=0x13，功能：显示字符串
! 输入：al=放置光标方式及规定属性。0x01表示使用bl中属性值，光标停在字符串结尾处；
!      es:bp 指向要显示的字符串起始位置。 cx=显示字符串个数； bh=显示页面号
!      bl=字符属性； dh=行号； dl=页号

	mov	ah,#0x03		! read cursor pos 读光标
	xor	bh,bh           ! 将bh置为0
	int	0x10
	
	mov	cx,#24          ! 24个字符
	mov	bx,#0x0007		! page 0, attribute 7 (normal) bh=0,页=0; bl=7,字符属性=7
	mov	bp,#msg1        ! es:bp指向要显示的字符串
	mov	ax,#0x1301		! write string, move cursor ah=0x13使用中断0x10功能号；al=0x01，使用bl中属性值
	int	0x10            ! 打开中断，串口打印字符串

! 上面使用中断0x10显示字符，首先使用ah=0x03功能获取光标位置以及行号列号，作为ah=0x13中断的入参；
! 而后使用ah=0x13中断将存在es:bp寄存器的字符串打印在串口屏幕，只要在使用中断时，将输入设定好即可。

! ok, we've written the message, now
! we want to load the system (at 0x10000) 将system模块加载到内存0x10000往后的120KB中

	mov	ax,#SYSSEG  !SYSSEG=0x10000
	mov	es,ax		! segment of 0x010000
	call	read_it
	call	kill_motor
```

此时整个操作系统的代码已经全部加载至内存。

确定一下根设备号

```assembly
! After that we check which root-device to use. If the device is
! defined (!= 0), nothing is done and the given device is used.
! Otherwise, either /dev/PS0 (2,28) or /dev/at0 (2,8), depending
! on the number of sectors that the BIOS reports currently.
! 确定根文件系统设备号并保存其设备号于root_dev
! Linux中，软驱的主设备号是2，次设备号=type*4+nr，其中nr为0-3分别对应软驱A、B、C和D；
! type是软驱类型（2->1.2MB或7->1.44MB）。
! 因为7*4+0=28，所以/dev/PS0 (2,28)指1.44MB A驱动器，其设备号是0x021c（2*256+28）
! 同理，/dev/at0 (2,8)值1.2MB A驱动器，设备号是0x0208

! 取上面获得的每磁道扇区数，如果sectors=15说明是1.2MB的驱动器；如果sectors=18说明是1.44M软驱（为什么？）；
! 因为是可引导的驱动器，所以肯定是A驱

	seg cs
	mov	ax,root_dev
	cmp	ax,#0
	jne	root_defined  ! 检查root_dev是否是空，如果否，则说明其已经存入根设备号，直接跳转后面
	seg cs            ! 此条指令表示下一条语句的操作数在cs段寄存器所指的段中如果以Masm语法写，
	                    seg cs和mov bx,sectors两句合起来，等价于mov bx, cs:[sectors], 这里使用了间接寻址方式。
                        重复一下前面的解释，mov [sectors],ax表示将ax中的内容存入ds:sectors内存单元，而mov cs:[sectors],ax强制以cs作为段地址寄存器，因此是将ax的内容存入cs:sectors内存单元
	mov	bx,sectors
	mov	ax,#0x0208		! /dev/ps0 - 1.2Mb
	cmp	bx,#15          ! 将sectors与15对比，如果相同，则ax=0x0208，最终赋值给root_defined
	je	root_defined
	mov	ax,#0x021c		! /dev/PS0 - 1.44Mb
	cmp	bx,#18          ! 将sectors与18对比，如果相同，则ax=0x021c，最终赋值给root_defined
	je	root_defined
undef_root:
	jmp undef_root
root_defined:         !将获取的驱动设备号存入root_dev
	seg cs
	mov	root_dev,ax
```

跳转到setup的第一条指令

```assembly
! after that (everyting loaded), we jump to
! the setup-routine loaded directly after
! the bootblock:

	jmpi	0,SETUPSEG       ! 跳转到setup程序开始处（即0x90200）执行setup程序
```



