---
title: "Linux内核源码学习——setup.s"
subtitle: "Linux内核源码学习"
layout: post
author: "Veronica"
header-style: text
tags:
  - Linux
---

##### 1.setup程序启动的第一件事

利用BIOS提供的中断服务程序从设备上提取内核运行所需的机器系统数据，如光标位置、显示页面等。并从中断向量0x41和0x46向量值所指的内存地址处获取硬盘参数表1和2. 

机器系统数据所占的内存空间为0x90000-0x901FD(510字节)

```assembly
! 提取机器系统数据
! 利用ROS BIOS中断读取机器系统数据，并将这些数据保存到0x90000开始的位置（0x90000-0x901FD）

! ①使用BIOS中断0x10功能号ah=0x03，读光标位置，并保存在内存0x90000处(2字节)。中断0x10功能号ah=0x03介绍：
! 输入：bh=页号
! 返回：ch=扫描开始线； cl=扫描结束线； dh=行号（0x00顶端）； dl=列号（0x00最左边）

	mov	ax,#INITSEG	! this is done in bootsect already, but... 0x9000
	mov	ds,ax
	mov	ah,#0x03	! read cursor pos 中断0x10功能号ah=0x03，读光标位置
	xor	bh,bh       ! 页号置0
	int	0x10		! save it in known place, con_init fetches
	mov	[0],dx		! it from 0x90000. 将光标行和列号保存在ds x 10H + 0

! Get memory size (extended mem, kB)
! ②利用BIOS中断0x15功能号 ah=0x88 取系统所含扩展内存大小并保存在内存0x90002处
! 中断返回值： ax= 从0x100000 （1M）处开始的扩展内存大小（KB）。若出错则CF置位，ax = 出错码

	mov	ah,#0x88
	int	0x15
	mov	[2],ax
```



##### 2.向32位模式转变

接下来，操作系统要使计算机在32位保护模式下工作。

**(1)关中断，并将system移动到内存地址起始位置0x00000**

关中断：将CPU的标志寄存器（EFLAGS）的IF（中断允许标志）置为0

```assembly
cli			! no interrupts allowed !禁止硬件中断
```

**(2)将位于0x10000的内核程序复制到内存地址的起始位置0x00000**

```assembly
! first we move the system to it's rightful place  将system移动到内存地址起始位置0x00000

	mov	ax,#0x0000
	cld			! 'direction'=0, movs moves forward  cld是清方向标志位，使DF=0（使一次计数+1，如果DF=1，则一次计数-1）
do_move:
	mov	es,ax		! destination segment
	add	ax,#0x1000
	cmp	ax,#0x9000
	jz	end_move
	mov	ds,ax		! source segment  ds:si=0x1000:0
	sub	di,di
	sub	si,si       ! destination segment es:di=0x0000:0
	mov 	cx,#0x8000      ！计数器，移动0x8000(0x10000-0x8fff)字
	rep
	movsw
	jmp	do_move
```

0x00000这个位置原来存放着由BIOS建立的中断向量表以及BIOS数据区，这个复制将把BIOS中断向量表和数据区完全覆盖，直到新的中断服务体系构建完成之前，操作系统都不具有处理中断的能力。（废除了16位的中断机制）

**(3)设置中断描述符表和全局描述符表**

GDT：全局描述符表，存放段寄存器内容的数组，用于进程切换，可以理解为进程的总目录表，其中存放每一个task的局部描述符表(LDT)地址和任务状态段(TSS)地址

GDTR：GDT基地址寄存器，标识GDT入口。在操作系统对GDT的初始化完成后，可以用LGDT指令将GDT基址加载进来。

IDT：中断描述符表，存有保护模式下所有中断服务程序的入口地址

IDTR：保存IDT的起始地址

```assembly
end_move:  ! ①加载中断描述符表和全局描述符表
	mov	ax,#SETUPSEG	! right, forgot this at first. didn't work :-)
	mov	ds,ax
	lidt	idt_48		! load idt with 0,0
	lgdt	gdt_48		! load gdt with whatever appropriate
```

(3)打开A20，实现32位寻址

32位寻址最大寻址空间为4GB

```assembly
! that was painless, now we enable A20 开启A20地址线，实现32位寻址

	call	empty_8042    ! 测试8042状态寄存器，等待输入缓冲器空，只有当输入缓冲器空才可以对其执行写命令
	mov	al,#0xD1		! command write 0xD1命令码——表示要写数据到8042的P2端口。P2端口的位1用于A20线的选通，数据要写到0x60口
	out	#0x64,al        ! 读端口用in, 写端口用out  IN AL,21H 表示从21H端口读取一字节数据到AL /  OUT 21H,AL 将AL的值写入21H端口
	call	empty_8042  ! 等待输入缓冲器空，看命令是否被接受
	mov	al,#0xDF		! A20 on
	out	#0x60,al
	call	empty_8042  ! 测试：若输入缓冲器为空，则表示A20线已选通
	

! 等待输入缓冲器为空
empty_8042:
	.word	0x00eb,0x00eb  !跳转到下一句，为了延时
	in	al,#0x64	! 8042 status port 读端口0x64到al-就是读8042的状态寄存器，（一个8bit的只读寄存器），bit_1为1时表示输入缓冲器满，为0时表示输入缓冲器空。要向8042写命令（通过0x64端口写入），必须当输入缓冲器为空时才可以。
	test	al,#2		! is input buffer full? 检测bit_1,如果为1，则跳转到empty_8042标号处继续检测，直到bit_1为0才返回
	jnz	empty_8042	! yes - loop
	ret
```

(4)建立保护模式下的中断机制：对可编程中断控制器8259A进行重新编程

```assembly
! ----ICW1
	mov	al,#0x11		! initialization sequence 
	out	#0x20,al		! send it to 8259A-1 ICW1主片端口地址0x20
	.word	0x00eb,0x00eb		! jmp $+2, jmp $+2 提供 14~20 个 CPU 时钟周期的延迟时间
	out	#0xA0,al		! and to 8259A-2 ICW2,从片端口地址0xA0
	.word	0x00eb,0x00eb
	! ----ICW2
	mov	al,#0x20		! start of hardware int's (0x20) 起始中断号0x20
	out	#0x21,al
	.word	0x00eb,0x00eb
	mov	al,#0x28		! start of hardware int's 2 (0x28) 起始中断号0x28
	out	#0xA1,al
	.word	0x00eb,0x00eb
	! ----ICW3
	mov	al,#0x04		! 8259-1 is master
	out	#0x21,al
	.word	0x00eb,0x00eb
	mov	al,#0x02		! 8259-2 is slave
	out	#0xA1,al
	.word	0x00eb,0x00eb
	! ----ICW4
	mov	al,#0x01		! 8086 mode for both
	out	#0x21,al
	.word	0x00eb,0x00eb
	out	#0xA1,al
	.word	0x00eb,0x00eb

	mov	al,#0xFF		! mask off all interrupts for now
	out	#0x21,al
	.word	0x00eb,0x00eb
	out	#0xA1,al
```

将CPU工作方式设为保护模式，将CR0寄存器第0位(PE)置为1，置1表示CPU工作在保护模式下，置0表示实模式

```assembly
mov	ax,#0x0001	! protected mode (PE) bit     //将cpu的工作方式设为保护模式，第0位为PE标志，置1时CPU工作在保护模式下，置0时为实模式
lmsw	ax		! This is it! lmsw指令仅仅加载CR0的低4位，由低到高分别是PE，MP，EM，TS
jmpi	0,8		! jmp offset 0 of segment 8 (cs)
```

这里重点讲一下 jmpi 0,8：

0表示段内偏移，8是保护模式下的段选择子，理解为二进制的1000。

1000的位0-1:表示内核特权级，Linux操作系统只用到两级——0级（内核级）和3级（用户级）。位2:0表示GDT，如果是1，则表示LDT。位3-15是描述符表项的索引，指出选择第几项描述符。

所以段选择子8(= 0000_0000_0000_1000b)表示请求特权级0、使用全局描述符表GDT中第1个段描述符项，该项是一个代码段描述符，指出代码段的基地址是0，又因为偏移值是0，所以这个跳转指令会跳转到0地址，即运行system模块。