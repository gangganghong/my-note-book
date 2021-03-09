# 怎么实现进程切换

## 是什么

进程是一个运行中的程序实体，拥有独立的地址空间和逻辑控制流。

```c
void sayHi()
{
  printf("%s\n", "Hello,World");
  return 0;
}
```

`sayHi`就是一个函数，它一旦运行起来，就是进程。

独立的逻辑控制流，是说这个进程就像独占一个CPU一样。每个进程使用CPU的时间不是连续的，但它们的指令运行却是前后衔接的，不会受到其他进程的指令对它的指令和数据大的更改。

## 运行起来

进程，这里是指用户进程（区别于内核），处在特权级3。内核在特权级0。CPU从通电运行起，就处在特权级0。要运行用户进程，需要从特权级0转移到特权级3。

CPU执行哪条指令，受`cs:eip`控制。有一个指令，既能够实现特权级从高向低转移，又能更新`cs:eip`。这个指令就是`iretd`。

执行第一个用户进程的方法是，把`cs、eip、ss、esp`等寄存器需要的值入栈，然后用`irted`出栈，用户进程就能运行起来了。示意代码如下。注意，下面的代码只是说明大概思路，不是可执行的代码。

```assembly
push		ss
push		esp
push		cs
push		eip

irted
```

## 停下来

操作系统要让一个CPU能运行多个进程。一个进程不能总是独占CPU，必须让它停下来，把CPU让给其他进程使用。

时钟中断每隔一段时间就会停止当前进程，转移到执行中断例程。

## 切换

在中断例程中，我们可以为当前进程A建立一个”快照“，选择运行进程B。

进程的”快照“，是把进程正在使用的数据、下一条要运行的指令等信息存储起来；然后选择另外一个进程，从存储设备中取出这个进程运行所需要的所有信息，最后运行这个进程。

进程正在使用的数据，全部都在寄存器中，把寄存器中的数据存起来，就是为进程建立了快照。

把数据存储到哪里呢？在汇编语言中，动态存储数据，我发现只有堆栈可以使用。进程的切换就是这样一个流程：

1. 进程A正在运行，时钟中断发生，执行中断例程。
2. 从TSS的`esp0`中获取进程A的堆栈栈顶，把`esp`指向这个栈顶。
3. 把寄存器中的值都压入进程A的堆栈中。
4. 把`esp`指向进程B的堆栈。调度程序就在这个步骤。
5. 把TSS的`esp0`指向进程B的堆栈的最高地址处加4个字节。下一次中断发生时，TSS获取的`esp0`的值就是在这里获取的。
6. B的堆栈出栈。
7. 使用`iretd`出栈`ss、esp、cs、eip`，开始执行进程B的指令。

## 堆栈转移

从上面的切换过程，很容易看出，需要转移堆栈。

第1步~~~第3步，堆栈从用户进程中的不知名堆栈转移到存储进程A数据的堆栈。这个堆栈，为进程建立快照使用。每个进程都有一个这样的堆栈。

在第4步前后，都有一次堆栈切换：

1. 前面，从A进程的堆栈切换到内核堆栈。进程调度程序很有可能会使用堆栈。如果仍然使用进程A的堆栈，会破坏A的堆栈，重新运行A时会有许多麻烦。
2. 进程调度结束后，已经选择了进程B，需要从堆栈中恢复B的数据，于是把`esp`指向B的堆栈。

进入内核后，我们切换过GDT和内核堆栈。当初我觉得没必要切换内核堆栈，现在终于发现了内核堆栈的作用。

那个内核堆栈是这样建立的：先用0填充一段内存空间，比如4KB，然后在下面设置一个标号，这个标号就是内核堆栈。代码如下：

```assembly
StatckStrace		1024  resp		0
TopStack:
```

# 理解切换进程的优雅方式

## 代码

```assembly
ALIGN	16
hwint00:		; Interrupt routine for irq 0 (the clock).
	call	save

	mov	al, EOI			; `. reenable
	out	INT_M_CTL, al		; /  master 8259

	sti
	push	0					
	call	clock_handler
	add	esp, 4
	cli

	ret
	
	
save:
        pushad          ; `.
        push    ds      ;  |
        push    es      ;  | 保存原寄存器值
        push    fs      ;  |
        push    gs      ; /
        mov     dx, ss
        mov     ds, dx
        mov     es, dx

        mov     eax, esp                    ;eax = 进程表起始地址

        inc     dword [k_reenter]           ;k_reenter++;
        cmp     dword [k_reenter], 0        ;if(k_reenter ==0)
        jne     .1                          ;{
        mov     esp, StackTop               ;  mov esp, StackTop <--切换到内核栈
        push    restart                     ;  push restart
        jmp     [eax + RETADR - P_STACKBASE];  return;
.1:                                         ;} else { 已经在内核栈，不需要再切换
        push    restart_reenter             ;  push restart_reenter
        jmp     [eax + RETADR - P_STACKBASE];  return;
                                            ;}

; ====================================================================================
;                                   restart
; ====================================================================================
restart:
	mov	esp, [p_proc_ready]
	lldt	[esp + P_LDT_SEL] 
	lea	eax, [esp + P_STACKTOP]
	mov	dword [tss + TSS3_S_SP0], eax
restart_reenter:
	dec	dword [k_reenter]
	pop	gs
	pop	fs
	pop	es
	pop	ds
	popad
	add	esp, 4
	iretd
```

进程A正在运行，时钟中断发生时，进入中断例程`hwint00`，将`ss、esp、eflags、cs、eip`依次入栈。这个栈是进程A的堆栈。

## save

`call	save`，将`save`的下一条指令`mov	al, EOI`的内存地址入栈，正好是进程A的堆栈的`retaddr`这个位置。

```c
typedef struct s_stackframe {	/* proc_ptr points here				↑ Low			*/
	u32	gs;		/* ┓						│			*/
	u32	fs;		/* ┃						│			*/
	u32	es;		/* ┃						│			*/
	u32	ds;		/* ┃						│			*/
	u32	edi;		/* ┃						│			*/
	u32	esi;		/* ┣ pushed by save()				│			*/
	u32	ebp;		/* ┃						│			*/
	u32	kernel_esp;	/* <- 'popad' will ignore it			│			*/
	u32	ebx;		/* ┃						↑栈从高地址往低地址增长*/		
	u32	edx;		/* ┃						│			*/
	u32	ecx;		/* ┃						│			*/
	u32	eax;		/* ┛						│			*/
	u32	retaddr;	/* return address for assembly code save()	│			*/
	u32	eip;		/*  ┓						│			*/
	u32	cs;		/*  ┃						│			*/
	u32	eflags;		/*  ┣ these are pushed by CPU during interrupt	│			*/
	u32	esp;		/*  ┃						│			*/
	u32	ss;		/*  ┛						┷High			*/
}STACK_FRAME;
```

`s_stackframe`的`retaddr`元素存储的就是`save`的下一条指令的内存地址，也就是存储在`eip`寄存器中的数据。这个常识会阻碍理解`retaddr`的正确含义。栈中没有规定哪个元素一定必须是哪个寄存器中的数据，出栈时只会根据栈中元素的顺序将它们一一更新到相应的寄存器中。

`eax + RETADR - P_STACKBASE`。`P_STACKBASE`是进程表中栈S的初始地址，`RETADR`是`retaddr`相对于S的初始地址的地址，`RETADR - P_STACKBASE`是`retaddr`在S中的偏移量，`eax + RETADR - P_STACKBASE`是`retaddr`在整个内存中的内存地址。

`jmp     [eax + RETADR - P_STACKBASE]`，会跳转到`call save`的下一条指令`mov	al, EOI`的内存地址处并且执行。

在`save`中，入栈`restart`或`restart_reenter`。注意，在`save`中只入栈，并不执行入栈的这两段代码。

<u>`save`是一个函数，却没有使用`ret`返回`save`的下一条指令并执行。这是因为，在`save`的末尾，堆栈中已经入栈了许多其他数据，使用`ret`出栈的并不是`call save`的下一条指令。</u>

```assembly
mov	al, EOI			; `. reenable
out	INT_M_CTL, al		; /  master 8259
```

让时钟中断可以发生多次。

```assembly
sti
push	0					
call	clock_handler
add	esp, 4
cli
```

进入时钟中断例程后，CPU会自动关闭中断，所以需要手动使用`sti`打开中断。

进程调度算法在`call	clock_handler`中，实质是选择下一个运行的进程的堆栈。

若在`save`中入栈的是``restart_reenter``，`hwint00`的末尾的`ret`指令，从进程A的堆栈中出栈的数据是`restart_reenter`的内存地址，这个内存地址更新到`cs:eip`中的`eip`。那么，下一条要执行的指令是`restart_reenter`，从而会执行`restart_reenter`这个函数。

`restart_reenter`的末尾是`irted`。这个指令会出栈进程A的堆栈中的四个数据，正好恢复执行被这个重入中断打断的上层中断。

若在`save`中入栈的是`restart`，`hwint00`的末尾的`ret`指令，从进程A的堆栈中出栈的数据是`restart`的内存地址，这个内存地址更新到`cs:eip`中的`eip`。那么，下一条要执行的指令是`restart`，从而会执行`restart`这个函数。

再看一次`restart`这个函数。

```assembly
restart:
	mov	esp, [p_proc_ready]
	lldt	[esp + P_LDT_SEL] 
	lea	eax, [esp + P_STACKTOP]
	mov	dword [tss + TSS3_S_SP0], eax
restart_reenter:
	dec	dword [k_reenter]
	pop	gs
	pop	fs
	pop	es
	pop	ds
	popad
	add	esp, 4
	iretd
```

在`restart`的末尾，并没有`ret、iretd`等这类指令。这意味着，`restart`函数的函数体还包括`restart_reenter`的函数体。

理解了这一点，一切疑团都解开了！

如果不是重入中断，那么，处理流程是：把esp指向要下一个要运行的进程的堆栈，并且把TSS的sp0指向这个进程的堆栈，供下次中断保存这个进程时使用。

## 疑问

理解不了打开中断和关闭中断的位置。





# 怎么实现itoa---把整数转换成十六进制形式

`itoa`把一个整型数转换成十六进制数，整型数前面的0要去掉。例如：`016`，转换后，结果是，`0x10`。

实现流程如下：

1. 用`str`存储转换后的结果，要被转换的数字是`num`。
2. 先给`str`加上`0x`。
3. 如果`num`的值是0，函数直接返回值：`0x0`。
4. 设置`flag`为false，标识一个数字是不是整数高位的0。
5. 建立一个循环，控制变量`i`的值是28，循环继续执行的条件是`i>=0`，`i`不断自减，减幅是4。如此，总计将会循环8次。
   1. 循环体如下。
   2. `num`左移`i`位，结果为`n`，`n`与`0xff`进行`与`运算，结果是`ch`。
   3. `flag | ch`
      1. 上面的结果是false，进入下次循环。
      2. 上面的结果是true，
         1. 设置`flag`的值是true
         2. 设置`newCh = ch + '0'`
         3. 比较`ch`和数字9的大小
            1. 比9大，设置`newCh = newCh + 7`。
            2. 比9小，什么也不做。
         4. 把`newCh`拼接到`str`。
6. `str`就是最终结果：由`num`转换成的十六进制数。

# 计算机在线课程

作者：haoxiang lin
链接：https://www.zhihu.com/question/29224038/answer/93653123
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



Update：
鉴于很多人问没有视频怎么破=。=这里再推荐一个清华贵系的OS课
[操作系统-学堂在线慕课(MOOC)平台](https://link.zhihu.com/?target=http%3A//www.xuetangx.com/courses/course-v1%3ATsinghuaX%2B30240243X_tv%2B2015_T1/about)
这个就有完整视频讲解，包括所有的lab，lab做完后也是自己完成了一个ucore os，而且文档都是中文的，觉得MIT的wiki看不懂的就上上这个吧=。=不过这门课当然是没有6.828的完整了，建议两个结合着来，看THU的视频，然后把6.828的paper，note看完，lab的话可以做完ucore的可以再参考下xv6的实现。

至于数据库和分布式的课，我觉得完全不需要视频。。。。

=========================分割线=========================================

MIT 6.828 经典的OS的神课，所有的lab，note和timeline在主页里都有，跟完课程就自己写完了一个简单的OS，大名鼎鼎的xv6,jos

[6.828 / Fall 2014](https://link.zhihu.com/?target=https%3A//pdos.csail.mit.edu/6.828/2014/schedule.html)

MIT 6.830 经典的数据库的课，所有的lab，note，还有需要读的paper，跟完就自己写了一个RDBMS

[6.830/6.814: Database Systems](https://link.zhihu.com/?target=http%3A//db.csail.mit.edu/6.830/)

MIT 6.824 经典的分布式系统的课，同样，跟完就自己完成一个Golang实现的分布式K/V databases

[6.824 Schedule: Spring 2015](https://link.zhihu.com/?target=http%3A//nil.csail.mit.edu/6.824/2015/schedule.html)