# 系统调用--ticks

## 笔记

```c
/*======================================================================*
                            kernel_main
 *======================================================================*/
PUBLIC int kernel_main()
{
	disp_str("-----\"kernel_main\" begins-----\n");

	TASK*		p_task		= task_table;
	PROCESS*	p_proc		= proc_table;
	char*		p_task_stack	= task_stack + STACK_SIZE_TOTAL;
	u16		selector_ldt	= SELECTOR_LDT_FIRST;
	int i;
	for (i = 0; i < NR_TASKS; i++) {
		strcpy(p_proc->p_name, p_task->name);	// name of the process
		p_proc->pid = i;			// pid

		p_proc->ldt_sel = selector_ldt;

		memcpy(&p_proc->ldts[0], &gdt[SELECTOR_KERNEL_CS >> 3],
		       sizeof(DESCRIPTOR));
		p_proc->ldts[0].attr1 = DA_C | PRIVILEGE_TASK << 5;
		memcpy(&p_proc->ldts[1], &gdt[SELECTOR_KERNEL_DS >> 3],
		       sizeof(DESCRIPTOR));
		p_proc->ldts[1].attr1 = DA_DRW | PRIVILEGE_TASK << 5;
		p_proc->regs.cs	= ((8 * 0) & SA_RPL_MASK & SA_TI_MASK)
			| SA_TIL | RPL_TASK;
		p_proc->regs.ds	= ((8 * 1) & SA_RPL_MASK & SA_TI_MASK)
			| SA_TIL | RPL_TASK;
		p_proc->regs.es	= ((8 * 1) & SA_RPL_MASK & SA_TI_MASK)
			| SA_TIL | RPL_TASK;
		p_proc->regs.fs	= ((8 * 1) & SA_RPL_MASK & SA_TI_MASK)
			| SA_TIL | RPL_TASK;
		p_proc->regs.ss	= ((8 * 1) & SA_RPL_MASK & SA_TI_MASK)
			| SA_TIL | RPL_TASK;
		p_proc->regs.gs	= (SELECTOR_KERNEL_GS & SA_RPL_MASK)
			| RPL_TASK;

		p_proc->regs.eip = (u32)p_task->initial_eip;
		p_proc->regs.esp = (u32)p_task_stack;
		p_proc->regs.eflags = 0x1202; /* IF=1, IOPL=1 */

		p_task_stack -= p_task->stacksize;
		p_proc++;
		p_task++;
		selector_ldt += 1 << 3;
	}

	k_reenter = 0;
	ticks = 0;

	p_proc_ready	= proc_table;

        put_irq_handler(CLOCK_IRQ, clock_handler); /* 设定时钟中断处理程序 */
        enable_irq(CLOCK_IRQ);                     /* 让8259A可以接收时钟中断 */

	restart();

	while(1){}
}
```



重点：

```c
put_irq_handler(CLOCK_IRQ, clock_handler); /* 设定时钟中断处理程序 */
enable_irq(CLOCK_IRQ);                     /* 让8259A可以接收时钟中断 */
```



```c
/*======================================================================*
                           put_irq_handler
 *======================================================================*/
PUBLIC void put_irq_handler(int irq, irq_handler handler)
{
	disable_irq(irq);
	irq_table[irq] = handler;
}
```



```assembly
; ========================================================================
;                  void disable_irq(int irq);
; ========================================================================
; Disable an interrupt request line by setting an 8259 bit.
; Equivalent code:
;	if(irq < 8){
;		out_byte(INT_M_CTLMASK, in_byte(INT_M_CTLMASK) | (1 << irq));
;	}
;	else{
;		out_byte(INT_S_CTLMASK, in_byte(INT_S_CTLMASK) | (1 << irq));
;	}
disable_irq:
        mov     ecx, [esp + 4]          ; irq
        pushf
        cli
        mov     ah, 1
        rol     ah, cl                  ; ah = (1 << (irq % 8))
        cmp     cl, 8
        jae     disable_8               ; disable irq >= 8 at the slave 8259
disable_0:
        in      al, INT_M_CTLMASK
        test    al, ah
        jnz     dis_already             ; already disabled?
        or      al, ah
        out     INT_M_CTLMASK, al       ; set bit at master 8259
        popf
        mov     eax, 1                  ; disabled by this function
        ret
disable_8:
        in      al, INT_S_CTLMASK
        test    al, ah
        jnz     dis_already             ; already disabled?
        or      al, ah
        out     INT_S_CTLMASK, al       ; set bit at slave 8259
        popf
        mov     eax, 1                  ; disabled by this function
        ret
dis_already:
        popf
        xor     eax, eax                ; already disabled
        ret

; ========================================================================
;                  void enable_irq(int irq);
; ========================================================================
; Enable an interrupt request line by clearing an 8259 bit.
; Equivalent code:
;       if(irq < 8){
;               out_byte(INT_M_CTLMASK, in_byte(INT_M_CTLMASK) & ~(1 << irq));
;       }
;       else{
;               out_byte(INT_S_CTLMASK, in_byte(INT_S_CTLMASK) & ~(1 << irq));
;       }
;
enable_irq:
        mov     ecx, [esp + 4]          ; irq
        pushf
        cli
        mov     ah, ~1
        rol     ah, cl                  ; ah = ~(1 << (irq % 8))
        cmp     cl, 8
        jae     enable_8                ; enable irq >= 8 at the slave 8259
enable_0:
        in      al, INT_M_CTLMASK
        and     al, ah
        out     INT_M_CTLMASK, al       ; clear bit at master 8259
        popf
        ret
enable_8:
        in      al, INT_S_CTLMASK
        and     al, ah
        out     INT_S_CTLMASK, al       ; clear bit at slave 8259
        popf
        ret
```



上面的代码，都看不懂。

## 其他

1秒(s) =1000 毫秒(ms) = 1,000,000 微秒(μs) = 1,000,000,000 纳秒(ns) = 1,000,000,000,000 皮秒(ps)=1,000,000,000,000,000飞秒(fs)=1,000,000,000,000,000,000仄秒(zs) =1,000,000,000,000,,000,000,000幺秒(ys)1,000,000,000,000,000,000,000,000渺秒(as)

CLK 引脚上的时钟脉冲信号是计数器的工作频率节拍，三个计数器的工作频率均是 1.19318MHz，即

一秒内会有 11 93180 次脉冲信号。每发生一次时钟脉冲信号，计数器就会将计数值减 ，也就是 秒内会

将计数值减 1193180 。当计数值递减为 时，计数器就会通过 OUT 引脚发出一个输出信号，此输出

信号用于向处理器发出时钟中断信号。一秒内会发出多少个输出信号，取决于计数值变成 的速度，也就

是取决于计数初始值是多少。默认情况下计数器。的初值寄存器值是 ，即表示 65536 。计数值从 65536

变成 需要修改 65536 次，所以，一秒内发输出信号的次数为 1193180/65536 ，约等于 18.206 ，即一秒内

发的输出信号次数为 18.206 次，时钟中断信号的频率为 18.206险。 1000 毫秒 ( 1193180/65536 ）约等于

54.925 ，这样相当于每隔 55 毫秒就发一次中断。这也解释了前面所讲的两个中断例子中，中断处理程序

输出字符为什么显得有些慢，当然大伙不做实验的话是看不到的，我当时没说而已。不过没关系，本节的

目的就是重新设定中断发生的频率，让中断发生得快一些，后面有咱们大展身手的机会。



1秒钟内，计数器会发生11 93180变化（减小）。当计数器初始值变为0时，会发出一个输出信号。

注意，发出输出信号时，会产生时间中断。

1秒钟内，计数器会发生11 93180变化（减小），能将初始值变成0多少次（多少次时钟中断）？ 11 93180 / 初始值。

每次时钟中断多少毫秒？1000毫秒 / 1000毫秒所发生的时钟中断次数 = 1000毫秒 / (11 93180 / 初始值)。

10毫秒产生一次时钟中断，1秒钟产生100次时钟中断。

11 93180 / 初始值 = 100 ===> 初始值 = 11 93180 / 100。

## 临时笔记

internal keyboard buffer full, ignoring scancode

### bochs创建虚拟硬盘

```shell
[root@localhost a]# bximage
========================================================================
                                bximage
  Disk Image Creation / Conversion / Resize and Commit Tool for Bochs
         $Id: bximage.cc 13481 2018-03-30 21:04:04Z vruppert $
========================================================================

1. Create new floppy or hard disk image
2. Convert hard disk image to other format (mode)
3. Resize hard disk image
4. Commit 'undoable' redolog to base image
5. Disk image info

0. Quit

Please choose one [0] 1

Create image

Do you want to create a floppy disk image or a hard disk image?
Please type hd or fd. [hd] hd

What kind of image should I create?
Please type flat, sparse, growing, vpc or vmware4. [flat]

Choose the size of hard disk sectors.
Please type 512, 1024 or 4096. [512]

Enter the hard disk size in megabytes, between 10 and 8257535
[10] 80

What should be the name of the image?
[c.img] 80m.img

Creating hard disk image '80m.img' with CHS=162/16/63 (sector size = 512)

The following line should appear in your bochsrc:
  ata0-master: type=disk, path="80m.img", mode=flat
```



## 硬盘分区

子扩展分区是在总扩展分区中创建的，子扩展分区的偏移扇区理应以总扩展分区的绝对扇区 LBA 地址为基准，因此，＂子扩展分区的绝对扇区 LBA 地址＝总扩展分区绝对扇区 LBA 地址＋子扩展分区的偏移扇区”。

逻辑分区是在子扩展分区中创建的，逻辑分区的偏移扇区理应以子扩展分区的绝对扇区 LBA 地址为基准，因此，“逻辑分区的绝对扇区 LBA 地址＝子扩展分区绝对扇区 LBA 地址＋逻辑分区偏移扇区飞这里的子扩展分区就是当前子扩展分区。







## 资源

《计算机系统要素》官方网站和书中工具下载地址

https://www.nand2tetris.org/software

谭光志说他一个月就做完了这本书中的所有试验（操作系统内核、编译器），我花了不止一个月还没有弄懂，我学习能力太差了吗？

### 工具--bochs

下载地址：

https://sourceforge.net/projects/bochs/files/bochs/2.6.11/

Bochs User Manual

http://bochs.sourceforge.net/cgi-bin/topper.pl?name=New+Bochs+Documentation&url=http://bochs.sourceforge.net/doc/docbook/user/index.html

Bochs 官网

http://bochs.sourceforge.net/getcurrent.html

### bochs和gdb调试C语言

#### 安装bocsh

```shell
./configure --prefix=/home/cg/tools/bochs-2.6.11 --enable-plugins   --enable-x86-64   --enable-cpp  --enable-disasm   --enable-gdb-stub --enable-x86-debugger --enable-e1000 
make
make install
```



### shell 方法



#### 错误

1. 

```shell
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LANG = "zh_CN.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to the standard locale ("C").
```

解决：

```shell
vi ~/.bashrc
export LANGUAGE="en_US.UTF-8"
export LANG=en_US:zh_CN.UTF-8 
export LC_ALL=C
source ~/.bashrc
```



```
========================================================================
00000000000i[      ] BXSHARE not set. using compile time default '/home/cg/tools/bochs-2.6.11/share/bochs'
00000000000i[      ] reading configuration from bochsrc
00000000000p[      ] >>PANIC<	< bochsrc:34: gdbstub directive malformed.
00000000000e[SIM   ] notify called, but no bxevent_callback function is registered
00000000000e[SIM   ] notify called, but no bxevent_callback function is registered
========================================================================
Bochs is exiting with the following message:
[      ] bochsrc:34: gdbstub directive malformed.
========================================================================
00000000000i[SIM   ] quit_sim called with exit code 1
```





