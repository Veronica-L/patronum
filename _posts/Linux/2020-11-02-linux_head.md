---
title: "Linux内核源码学习——head.s"
subtitle: "Linux内核源码学习"
layout: post
author: "Veronica"
header-style: text
tags:
  - Linux
---

##### head程序执行的整体策略:

bootsect是加载到0x07c00, setup是加载到0x90200。

而head的加载方式是：先将head.s汇编成目标代码，将C语言编写的内核程序编译成目标代码，然后链接成system模块。所以system模块既有内核程序，也有head程序。

setup将system模块复制到0x00000位置，因为head在system模块的前面部分，所以head程序就在0x00000这个位置。head程序占有25KB+184B的空间。head程序后面就是main函数。

head的工作：用程序自身的代码在程序自身所在的内存创建**内核分页机制**。即在0x00000创建了页目录表、页表、缓冲区、GDT、IDT。并覆盖已执行的代码。这意味着head将自己废弃，即将执行main函数。

#####  (1)将DS,ES,FS和GS从实模式转变为保护模式

```assembly
.text
.globl _idt,_gdt,_pg_dir,_tmp_floppy_area
_pg_dir:  #标识内核分页机制完成后的内核起始地址(物理内存的起始地址0x00000)，head在此处建立页目录表
startup_32:
  #ds,es,fs,gs值都为0x10
  #因为处理器已经工作在保护模式下，所以这些段寄存器都表示段选择子。0x10 写成16位二进制形式为0b0000 0000 0001 0000，所以值为该数的段选择子：请求特权级为 0（RPL=00）、所指向的描述符存放在GDT（TI=0）、所指向的描述符索引为2（DI=0000 000000010）
	movl $0x10,%eax
	mov %ax,%ds
	mov %ax,%es
	mov %ax,%fs
	mov %ax,%gs
	
	#lss:将操作数的值传送给指定寄存器ss:esp
	#stack_start定义在kenel/sched.c文件中
	/*long user_stack [ PAGE_SIZE>>2 ] ;
		struct {
		long * a;
		short b;
		} stack_start = { & user_stack [PAGE_SIZE>>2] , 0x10 };*/
	#将结构体stack_start的值传送到ss:esp，令 ss=0x10（段选择子）和 esp=&user_stack [PAGE_SIZE>>2]
	lss _stack_start,%esp
	
	call setup_idt #设置IDT
	call setup_gdt
	movl $0x10,%eax		# reload all the segment registers
	mov %ax,%ds		# after changing gdt. CS was already
	mov %ax,%es		# reloaded in 'setup_gdt'
	mov %ax,%fs
	mov %ax,%gs
	lss _stack_start,%esp
	xorl %eax,%eax
1:	incl %eax		# check that A20 really IS enabled
	movl %eax,0x000000	# loop forever if it isn't
	cmpl %eax,0x100000
	je 1b
```



##### (2)设置IDT

```assembly
setup_idt:
	lea ignore_int,%edx #先让所有的中断描述符默认指向ignore_int
	movl $0x00080000,%eax  #这里的8看成1000，这个值会在初始化IDT的时候用到
	movw %dx,%ax		/* selector = 0x0008 = cs */
	movw $0x8E00,%dx	/* interrupt gate - dpl=0, present */

	lea _idt,%edi
	mov $256,%ecx
rp_sidt:
	movl %eax,(%edi)
	movl %edx,4(%edi)
	addl $8,%edi
	dec %ecx
	jne rp_sidt
	lidt idt_descr
	ret
```

中断描述符为64位，包含0-15+48-63组合成32位的中断服务程序的段内偏移地址。16-31位为段选择符，定位中断服务程序所在段，47:段存在标志，45-46:特权级标志，40-43:段描述符类型标志

##### (3)设置GDT

```assembly
setup_gdt:
	lgdt gdt_descr
	ret
.align 2
.word 0
idt_descr:
	.word 256*8-1		# idt contains 256 entries
	.long _idt
.align 2
.word 0
gdt_descr:
	.word 256*8-1		# so does gdt (not that that's any
	.long _gdt		# magic number, but it works for me :^)

	.align 3
_idt:	.fill 256,8,0		# idt is uninitialized

_gdt:	.quad 0x0000000000000000	/* NULL descriptor */
	.quad 0x00c09a0000000fff	/* 16Mb */
	.quad 0x00c0920000000fff	/* 16Mb */
	.quad 0x0000000000000000	/* TEMPORARY - don't use */
	.fill 252,8,0			/* space for LDT's and TSS's etc */
```

GDT 共设置了 4 个项，余下的 252 个项初始化为 0

![gdt.png](https://i.loli.net/2020/11/02/sZ7GkDIoz9hQUuw.png)



##### (4)重新设置ds,es,fs,gs,ss

因为它们所指向的原描述符所指向的段的段限长为 8MB，所以当访问 8MB 以上的地址空间时，将会产生段限长超限报警。为了防止这类可能发生的情况，在这里需要对段寄存器（段选择子）重新设置：

```assembly
movl $0x10,%eax		# reload all the segment registers
	mov %ax,%ds		# after changing gdt. CS was already
	mov %ax,%es		# reloaded in 'setup_gdt'
	mov %ax,%fs
	mov %ax,%gs
	lss _stack_start,%esp
```

##### (5)检查A20是否打开

A20如果没有打开，计算机处于20位的寻址模式，超过0xFFFFF寻址必然会回滚，如0x100000会回滚到0x000000，也就是说地址0x100000存储的值会和0x000000一样。

所以通过在内存0x000000位置写入数据并和0x100000处比较来检查A20是否被打开。如果一直相同的话，就一直比较下去，即死循环、死机，表示 A20 线没有选通，结果内核就不能够使用 1MB 以上内存。

```assembly
	xorl %eax,%eax
1:	incl %eax		# check that A20 really IS enabled
	movl %eax,0x000000	# loop forever if it isn't
	cmpl %eax,0x100000
	je 1b
```

##### (6)检查x87协处理器是否存在

为了弥补 x86 系列在进行浮点计算时的不足，Intel 于 1980 年推出了 x87 系列数学协处理器，那时是一个外置的、可选的芯片。1989 年，Intel 发布了 486 处理器。自从 486 开始，以后的 CPU 一般都内置了协处理器。这样，对于 486 以前的计算机而言，操作系统检验 x87 协处理器是否存在就非常有必要了。

```assembly
/*
 * NOTE! 486 should set bit 16, to check for write-protect in supervisor
 * mode. Then it would be unnecessary with the "verify_area()"-calls.
 * 486 users probably want to set the NE (#5) bit also, so as to use
 * int 16 for math errors.
 */
	movl %cr0,%eax		# check math chip
	andl $0x80000011,%eax	# Save PG,PE,ET
/* "orl $0x10020,%eax" here for 486 might be good */
	orl $2,%eax		# set MP
	movl %eax,%cr0
	call check_x87
	jmp after_page_tables

/*
 * We depend on ET to be correct. This checks for 287/387.
 */
check_x87:
	fninit
	fstsw %ax
	cmpb $0,%al
	je 1f			/* no coprocessor: have to set bits */
	movl %cr0,%eax
	xorl $6,%eax		/* reset MP, set EM */
	movl %eax,%cr0
	ret
.align 2
1:	.byte 0xDB,0xE4		/* fsetpm for 287, ignored by 387 */
	ret
```



##### (7)构建分页管理机制

```assembly
startup_32:
	……
	call check_x87
	jmp after_page_tables

after_page_tables:
	#先将main函数参数，L6标号和main函数入口地址压栈
	pushl $0		# These are the parameters to main :-)
	pushl $0
	pushl $0
	pushl $L6		# return address for main, if it decides to.
	pushl $_main
	jmp setup_paging 
L6:
	jmp L6			# main should never return here, but
				# just in case, we know what happens.
```

完成后跳转到setup_paging，开始创建分页机制

**1.先将页目录表和4个页表放在物理内存的起始位置，从内存起始位置开始的 5页空间内从全部清零，每页4KB。**

```assembly
.align 2
setup_paging:
	movl $1024*5,%ecx		/* 5 pages - pg_dir+4 page tables */
	xorl %eax,%eax
	xorl %edi,%edi			/* pg_dir is at 0x000 */
	cld;rep;stosl
```

`stosl`：每次保存的是 4 个字节。

- ecx 控制循环次数

- 每次循环将 eax 的值保存到 es:edi （es 为段选择子）指向的内存

- 若 EFLAGS 中的方向标志位 DF=0 （使用 cld 指令），则 edi 自增 4 （因为比较的是 Long，所以递增 4）；若 DF=1（使用 std 指令），则 edi 自减 4

- rep 表示当 ecx>0 时，循环继续；反之停止

- 在这个程序中，每循环 1024 次，清零的内存范围是 1024*4=4096 字节，恰好是一个页。

  

**2.设置页目录表的前四项，使之分别指向4个页表**

```assembly
/*
 * I put the kernel page tables right after the page directory,
 * using 4 of them to span 16 Mb of physical memory. People with
 * more than 16MB will have to expand this.
 */
.org 0x1000
pg0:

.org 0x2000
pg1:

.org 0x3000
pg2:

.org 0x4000
pg3:

setup_paging:
	movl $1024*5,%ecx		/* 5 pages - pg_dir+4 page tables */
	xorl %eax,%eax
	xorl %edi,%edi			/* pg_dir is at 0x000 */
	cld;rep;stosl
	/*
		7看成二进制的 111，代表页属性，u/s、r/w、present
		111 代表：用户 u、读写 rw、存在 p
		000 代表：内核 s、只读 r、不存在
	*/
	movl $pg0+7,_pg_dir		/* set present bit/user r/w */
	movl $pg1+7,_pg_dir+4		/*  --------- " " --------- */
	movl $pg2+7,_pg_dir+8		/*  --------- " " --------- */
	movl $pg3+7,_pg_dir+12		/*  --------- " " --------- */
```



**3.页表填充**

从高地址向低地址填写4个页表

```assembly
.align 2
setup_paging:
	……
	movl $pg3+4092,%edi  #第4个页表的最后一个页表项的起始位置
	movl $0xfff007,%eax		/*  16Mb - 4096 + 7 (r/w user,p) */
	#存储到第 4 个页表的最后一个页表项的内容。这里 7 表示页属性，0xfff000 为该页表项所指向的页基址（也称为页号）。该地址刚好是16MB内存的最后一页的地址
	
	std
#eax（初始值为 0xfff007，7为页属性）递减0x1000 （4KB，一个页大小），
#edi（初始值为 $pg3+4092） 按 4 (std，表示 4 个字节)递减
#将 eax 内容（即页表项）存储到内存 edi 处
#这样直到循环结束，刚好能将 4 个页表填满。每个页表有4KB/4B=1024个页表项。4个页表支持的寻址范围为4 * 1024 *4KB = 16MB，恰好是 Linux 0.11 支持的寻址范围
1:	stosl			/* fill pages backwards - more efficient :-) */
	subl $0x1000,%eax
	jge 1b
```

![page.png](https://i.loli.net/2020/11/02/4I926meJgQHotp7.png)

**4.设置CR3和CR0**

**CR0:**选择微处理器的工作方式和存储器的管理模式。其中第31位是PG标志，是分页机制控制位。当CPU的CR0的第0位PE（保护模式）置为1时，可以设置PG位为开启。开启后，地址映射模式采取分页机制。当PE（保护模式）置为0时，设置PG位会发生异常。

**CR3:**页目录表基址寄存器，保存"页表目录"的起始物理地址，CR3 的高 20 位提供页表目录的基地址，低 12 位“不用”（这里的“不用”并不是指真的不用，而是从 CR3 取 32 位数据时，低 12 位全“取”为 0。当CR0的PG标志位置位时，CPU使用CR3指向的页目录表和页表进行虚拟地址到物理地址的映射。

```assembly
.align 2
setup_paging:
	……
	xorl %eax,%eax		/* pg_dir is at 0x0000 */
	movl %eax,%cr3		/* cr3 - page directory start 将CR3指向页目录表 0x0000就是页目录表的起始位置*/
	movl %cr0,%eax
	orl $0x80000000,%eax
	movl %eax,%cr0		/* set paging (PG) bit 启动PG标志置位*/
	ret			/* this also flushes prefetch-queue  -----去了main函数---*/
```

![system_model.png](https://i.loli.net/2020/11/02/VCcQdDKWG1sYhkf.png)

**5.跳转至main函数**

```assembly
.align 2
setup_paging:
	……
	ret			/* this also flushes prefetch-queue  -----去了main函数---*/
```

在之前，main函数被压入了栈顶。现在执行了ret，正好将压入的main函数的执行入口地址弹给了EIP。