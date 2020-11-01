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
```

