# 实现IPC

## 代码位置

/home/cg/os/pegasus-os/v28

## 代码阅读

### sys_printx

```shell
; =============================================================================
;                                 sys_call
; =============================================================================
sys_call:
        call    save

        sti
	push	esi

	push	dword [p_proc_ready]
	push	edx
	push	ecx
	push	ebx
        call    [sys_call_table + eax * 4]
	add	esp, 4 * 4

	pop	esi
        mov     [esi + EAXREG - P_STACKBASE], eax
        cli

        ret

```

`call    [sys_call_table + eax * 4]`调用下面的函数

```c
PUBLIC int sys_printx(int _unused1, int _unused2, char* s, struct proc* p_proc)
{
	const char * p;
	char ch;

	char reenter_err[] = "? k_reenter is incorrect for unknown reason";
	reenter_err[0] = MAG_CH_PANIC;

	/**
	 * @note Code in both Ring 0 and Ring 1~3 may invoke printx().
	 * If this happens in Ring 0, no linear-physical address mapping
	 * is needed.
	 *
	 * @attention The value of `k_reenter' is tricky here. When
	 *   -# printx() is called in Ring 0
	 *      - k_reenter > 0. When code in Ring 0 calls printx(),
	 *        an `interrupt re-enter' will occur (printx() generates
	 *        a software interrupt). Thus `k_reenter' will be increased
	 *        by `kernel.asm::save' and be greater than 0.
	 *   -# printx() is called in Ring 1~3
	 *      - k_reenter == 0.
	 */
	if (k_reenter == 0)  /* printx() called in Ring<1~3> */
		p = va2la(proc2pid(p_proc), s);
	else if (k_reenter > 0) /* printx() called in Ring<0> */
		p = s;
	else	/* this should NOT happen */
		p = reenter_err;

	/**
	 * @note if assertion fails in any TASK, the system will be halted;
	 * if it fails in a USER PROC, it'll return like any normal syscall
	 * does.
	 */
	if ((*p == MAG_CH_PANIC) ||
	    (*p == MAG_CH_ASSERT && p_proc_ready < &proc_table[NR_TASKS])) {
		disable_int();
		char * v = (char*)V_MEM_BASE;
		const char * q = p + 1; /* +1: skip the magic char */

		while (v < (char*)(V_MEM_BASE + V_MEM_SIZE)) {
			*v++ = *q++;
			*v++ = RED_CHAR;
			if (!*q) {
				while (((int)v - V_MEM_BASE) % (SCR_WIDTH * 16)) {
					/* *v++ = ' '; */
					v++;
					*v++ = GRAY_CHAR;
				}
				q = p + 1;
			}
		}

		__asm__ __volatile__("hlt");
	}

	while ((ch = *p++) != 0) {
		if (ch == MAG_CH_PANIC || ch == MAG_CH_ASSERT)
			continue; /* skip the magic char */

		out_char(tty_table[p_proc->nr_tty].p_console, ch);
	}

	return 0;
}
```

前提条件是：

1. `call    [sys_call_table + eax * 4]`调用函数`int sys_printx(int _unused1, int _unused2, char* s, struct proc* p_proc)`
2. sys_printx只有四个参数，在调用语句`call    [sys_call_table + eax * 4]`前入栈5个元素，如何保证最先入栈的元素`esi`不被误认为是sys_printx的参数？
3. 这段代码是摘录自一本书，是正确的。那么，该怎么理解调用这个函数的代码？
   1. 这个函数只有四个参数，却在调用语句前入栈5个元素？

回答是：在调用sys_printx前入栈5个元素，完全没有问题。仔细看看，在save中还入栈了十几个元素。对入栈的十几个元素没有疑问，怎么就对和参数挨着的第5个元素有疑问呢？

在函数内部，会自动从堆栈中获取正确的元素当作自己的参数。无论是C函数还是汇编函数，编写者（包括编译器）知道本函数有多少个参数，会自觉从堆栈中获取所需数量的元素当作参数。编写者知道自己只需要4个参数，不会从堆栈中获取5个元素当参数。

在C函数内部，会这样从堆栈中获取参数：

```assembly
[ebp + 0]					; 旧ebp
[ebp + 4]					; 调用函数的指令的下一条指令的地址
[ebp + 8]					; 第一个参数
[ebp + 12]				; 第二个参数
[ebp + 16]				; 第三个参数
[ebp + 20]				; 第四个参数
```

调用C函数的代码是这样的：

```assembly
push	esi

	push	dword [p_proc_ready]
	push	edx
	push	ecx
	push	ebx
  call    [sys_call_table + eax * 4]
```

按照上面的`Calling Conventions`时，能从堆栈中正确获取到数据吗？能。模拟执行一次，就会发现，能。

入栈了esi，会影响函数接收参数吗？不会。函数知道自己有4个参数，只会从堆栈中获取4个元素，根本不会理睬esi（中的值）。

没理解问题的实质，思维定势，害得我在这样简单的问题上浪费3个小时左右。



```shell
(gdb) p p
$2 = 0x37db0 "\003  assert(0) failed: file: kernel/clock.c, base_file: kernel/clock.c, ln24"

(gdb) p *q
$28 = 0 '\000'
```

sys_printx中

```c
if (!*q) {
				// value of SCR_WIDTH is 80.
				while (((int)v - V_MEM_BASE) % (SCR_WIDTH * 16)) {
					/* *v++ = ' '; */
					v++;
					*v++ = GRAY_CHAR;
				}
				q = p + 1;
			}
```

这段代码的效果是下图。

![image-20210430143046242](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210430143046242.png)



1. 当*q的值是空格时，`!*q`仍为0。
2. 当*q的值是空字符时，`!*q`的值是1。

怎么弄明白的？断点调试看效果。



最终效果是：

![image-20210430143959484](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210430143959484.png)



### assertion_failure

代码流程：

1. /home/cg/yuyuan-os/osfs08/a/lib/misc.c#assertion_failure
2. /home/cg/yuyuan-os/osfs08/a/include/proto.h#  #define	printl	printf
3. sys_call.asm#printx
4. tty.c#sys_printx(int _unused1, int _unused2, char* s, struct proc* p_proc)

`char* s`：

1. 是`printx`中的`mov	edx, [esp + 4]`
2. 是printf的buf
3. 是`assertion_failure(char *exp, char *file, char *base_file, int line)`中的
   1. `printl("%c  assert(%s) failed: file: %s, base_file: %s, ln%d",
      	       MAG_CH_ASSERT,
      	       exp, file, base_file, line);`的第一个参数。
4. 结论：`sys_printx`的`char* s,`是``printl`第一个参数用具体值替换后的字符串。
   1. `int printf(const char *fmt, ...)`
   2. `char* s,`是fmt被替换后的字符串。

### proc2pid

```shell
/Users/cg/data/code/os/yy-os/osfs08/a/include/proc.h:
   75: #define proc2pid(x) (x - proc_table)
```

### va2la

```c
// proc.c
PUBLIC void* va2la(int pid, void* va)
{
	struct proc* p = &proc_table[pid];

	u32 seg_base = ldt_seg_linear(p, INDEX_LDT_RW);
	u32 la = seg_base + (u32)va;

	// 初始化进程时，进程的代码段、数据段的地址空间的基地址都设置成0。
	// 也就是说，seg_base = 0。
	// 那么，la 应该等于 va。
	if (pid < NR_TASKS + NR_PROCS) {
		assert(la == (u32)va);
	}

	return (void*)la;
}
```

`p = va2la(proc2pid(p_proc), s);`：

1. `proc2pid(p_proc)`，进程在进程表中的索引。
2. 进程中变量的内存地址。
3. 目前，并没有必要这样做，因为，我根本没有使用虚拟地址。于上神的代码，目前也没有使用虚拟地址。
4. 我实现assert时，此处暂时不必区分用户进程和非用户进程。
5. 不，我实现assert时，要区分用户进程和非用户进程。不能太简化了。

### MESSAGE

```c
/**
 * MESSAGE mechanism is borrowed from MINIX
 */
struct mess1 {
	int m1i1;
	int m1i2;
	int m1i3;
	int m1i4;
};
struct mess2 {
	void* m2p1;
	void* m2p2;
	void* m2p3;
	void* m2p4;
};
struct mess3 {
	int	m3i1;
	int	m3i2;
	int	m3i3;
	int	m3i4;
	u64	m3l1;
	u64	m3l2;
	void*	m3p1;
	void*	m3p2;
};
typedef struct {
	int source;				// sender 的 pid
	int type;				// 1.mess1;2.mess2;3.mess3
	union {
		struct mess1 m1;
		struct mess2 m2;
		struct mess3 m3;
	} u;
} MESSAGE;
```

```c
#define	RETVAL		u.m3.m3i1
```



MESSAGE的一个具体数值：

```shell
(gdb) p sender
$9 = (struct proc *) 0x61a70 <proc_table+304>
(gdb) p *sender
$10 = {regs = {gs = 59, fs = 15, es = 15, ds = 15, edi = 3296919552, esi = 326190, ebp = 327436, kernel_esp = 400032, ebx = 1,
    edx = 327460, ecx = 1, eax = 1, retaddr = 198359, eip = 198451, cs = 7, eflags = 518, esp = 327392, ss = 15}, ldt_sel = 88, ldts = {{
      limit_low = 65535, base_low = 0, base_mid = 0 '\000', attr1 = 249 '\371', limit_high_attr2 = 207 '\317', base_high = 0 '\000'}, {
      limit_low = 65535, base_low = 0, base_mid = 0 '\000', attr1 = 243 '\363', limit_high_attr2 = 207 '\317', base_high = 0 '\000'}},
  ticks = 5, priority = 5, pid = 2, name = "TestA\000\000\000\350H\"\000\000\203\304\020", p_flags = 0, p_msg = 0x0, p_recvfrom = 25,
  p_sendto = 25, has_int_msg = 0, q_sending = 0x0, next_sending = 0x0, nr_tty = 0}
```

这种消息有什么用？连字符串都没有。

### phys_copy

呵呵。原来，`#define	phys_copy	memcpy`。

```c
phys_copy(va2la(dest, p_dest->p_msg),
			  va2la(proc2pid(sender), m),
			  sizeof(MESSAGE));
```

把MESSAGE类型数据从sender复制到dest，这就是所谓的消息传递。

### msg_send

```shell
/*****************************************************************************
 *                                msg_send
 *****************************************************************************/
/**
 * <Ring 0> Send a message to the dest proc. If dest is blocked waiting for
 * the message, copy the message to it and unblock dest. Otherwise the caller
 * will be blocked and appended to the dest's sending queue.
 * 
 * @param current  The caller, the sender.
 * @param dest     To whom the message is sent.
 * @param m        The message.
 * 
 * @return Zero if success.
 *****************************************************************************/
PRIVATE int msg_send(struct proc* current, int dest, MESSAGE* m)
{
	struct proc* sender = current;
	struct proc* p_dest = proc_table + dest; /* proc dest */

	assert(proc2pid(sender) != dest);

	/* check for deadlock here */
	if (deadlock(proc2pid(sender), dest)) {
		panic(">>DEADLOCK<< %s->%s", sender->name, p_dest->name);
	}

	if ((p_dest->p_flags & RECEIVING) && /* dest is waiting for the msg */
	    (p_dest->p_recvfrom == proc2pid(sender) ||
	     p_dest->p_recvfrom == ANY)) {
		assert(p_dest->p_msg);
		assert(m);

		phys_copy(va2la(dest, p_dest->p_msg),
			  va2la(proc2pid(sender), m),
			  sizeof(MESSAGE));
		p_dest->p_msg = 0;
		p_dest->p_flags &= ~RECEIVING; /* dest has received the msg */
		p_dest->p_recvfrom = NO_TASK;
		unblock(p_dest);

		assert(p_dest->p_flags == 0);
		assert(p_dest->p_msg == 0);
		assert(p_dest->p_recvfrom == NO_TASK);
		assert(p_dest->p_sendto == NO_TASK);
		assert(sender->p_flags == 0);
		assert(sender->p_msg == 0);
		assert(sender->p_recvfrom == NO_TASK);
		assert(sender->p_sendto == NO_TASK);
	}
	else { /* dest is not waiting for the msg */
		sender->p_flags |= SENDING;
		assert(sender->p_flags == SENDING);
		sender->p_sendto = dest;
		sender->p_msg = m;

		/* append to the sending queue */
		struct proc * p;
		if (p_dest->q_sending) {
			p = p_dest->q_sending;
			while (p->next_sending)
				p = p->next_sending;
			p->next_sending = sender;
		}
		else {
			p_dest->q_sending = sender;
		}
		sender->next_sending = 0;

		block(sender);

		assert(sender->p_flags == SENDING);
		assert(sender->p_msg != 0);
		assert(sender->p_recvfrom == NO_TASK);
		assert(sender->p_sendto == dest);
	}

	return 0;
}
```

#### 消息传递

```c
phys_copy(va2la(dest, p_dest->p_msg),
			  va2la(proc2pid(sender), m),
			  sizeof(MESSAGE));
```

把MESSAGE类型数据从sender复制到dest，这就是所谓的消息传递。

#### 进程表和IPC相关的成员

成功接收消息后，这些成员的值应该满足下面的要求：

```c
// p_flags，进程的运行状态：0.运行
assert(p_dest->p_flags == 0);
// p_msg是消息，数据类型是MESSAGE *。
// 比较神奇，对一个结构体能这样赋值吗？试试就知道了。
// 确实挺神奇的。p_dest->p_msg需要的值是内存地址，所以可以用数字。而且，只有使用0时没有警告，使用其他数字，都会产生警告。
// 详情见见<结构体的赋值是0>。
assert(p_dest->p_msg == 0);
assert(p_dest->p_recvfrom == NO_TASK);
assert(p_dest->p_sendto == NO_TASK);

assert(sender->p_flags == 0);
assert(sender->p_msg == 0);
assert(sender->p_recvfrom == NO_TASK);
assert(sender->p_sendto == NO_TASK);
```

##### ANY、NO_TASK

```c
#define ANY		(NR_TASKS + NR_PROCS + 10)		// 15
#define NO_TASK		(NR_TASKS + NR_PROCS + 20)	// 25
```

##### 结构体的赋值是0

```c
#include <stdio.h>

typedef struct{
        char name[20];
        int age;
}Person;

typedef struct{
        Person *p;
}T;

int main(int argc, char **argv)
{
        //Person jim = 0;
        //printf("jim.name = %s\n", jim.name);
        //printf("jim.age = %d\n", jim.age);

        Person *jim = (Person *)123;
        //printf("jim->name = %s\n", jim->name);
        //printf("jim->age = %d\n", jim->age);

        if(jim == 0){
                printf("jim = %d\n", 0);
        }

        T t;
        t.p = 1;
        if(t.p == 0){
                printf("t.p = %d\n", 0);
                printf("t.p = %d\n", t.p);
        }

        return 0;
}
```

执行结果是：

```shell
[root@localhost experiment]# gcc -o struct-demo struct-demo.c -g -m32
struct-demo.c: In function 'main':
struct-demo.c:27:6: warning: assignment to 'Person *' {aka 'struct <anonymous> *'} from 'int' makes pointer from integer without a cast [-Wint-conversion]
  t.p = 1;
      ^
```

gdb调试过程：

```shell
(gdb) b main
Breakpoint 1 at 0x80484be: file struct-demo.c, line 18.
(gdb) run struct-demo
Starting program: /home/cg/yuyuan-os/osfs08/a/experiment/struct-demo struct-demo

Breakpoint 1, main (argc=2, argv=0xffffd3f4) at struct-demo.c:18
18		Person *jim = (Person *)123;
(gdb) s
22		if(jim == 0){
(gdb) p jim
$1 = (Person *) 0x7b
(gdb) s
27		t.p = 1;
(gdb) s
28		if(t.p == 0){
(gdb) p t.p
$2 = (Person *) 0x1
```

证实了我的猜想，赋值给结构体的是数字是内存地址。更正一下，并不是给结构体赋值，而是给结构体类型的指针赋值。

不管是什么类型的指针，只要是指针，都可以把整数当内存地址A赋值给它。指针的类型，作用是，找到内存地址是A开头的那片内存，

1. 如果指针的类型是char，那么，那片内存只有一个字节。
2. 如果指针的类型是int，那么，那片内存只有4个字节。
3. 如果指针的类型是Person，那么，那片内存只有sizeof(Person)个字节。

##### p_dest->p_flags & RECEIVING

p_dest->p_flags &= ~RECEIVING;

p_dest->p_flags & ~RECEIVING & RECEIVING，结果是什么？

1. 符合结合律吗？不符合。



```c
/* Process */
#define SENDING   0x02	/* set when proc trying to send */
#define RECEIVING 0x04	/* set when proc trying to recv */
```



## 工具

### gdb

```shell
#以当前行为起点，打印源码
#从第2行开始，打印源码
list 2
(gdb) show listsize
Number of source lines gdb will list by default is 10.
set listsize 5          //设置打印的行数为 5 行。
set listsize 6          //设置打印的行数为 6 行。
(gdb) list +2           //从当前行向下偏移 2 行
(gdb) list -2            //从当前行向上偏移 2 行
(gdb) list 1 , 5            //打印从第一行开始，到第五行结束。
(gdb) list  , 5               //打印从当前行开始，到第五行结束。
(gdb) list func                 //以func函数所在行为中心行打印。
(gdb) list *0x1234             //以标记地址所在行为中心行打印。
```

### idea

激活

https://sunjs.com/article/detail/6713ef497b9447dcbbd6f0557774bb4f.html

## 总结

### 2021-04-29 21:16

前前后后耗费了3个小时左右。弄明白了上面的 sys_printx 问题。其实，真正弄明白这个问题只花了几秒钟。真的，一点都不夸张，确实是在最后几秒突然想清楚了。

我小心翼翼地在一个微信群提问，碰碰运气呗，只有一个人回答，可答案完全不正确。

我翻看《一个操作系统的实现》、《汇编语言程序设计》，看OSDV的"Calling Conventions"，都没有找到相关内容。网络，不必看了。汇编相关的问题，不流行。

遇到这个问题，我忽然发现我理解不了之前理解了的问题，例如：进程表的寄存器的保存与弹出。

遇到问题，要有条理地想，越是没有头绪的问题，越要如此，要提出假设，而不要胡乱瞎找各种资料，一直盯着代码看。

### 2021-04-30 14:51

耗费50分钟。基本看明白了assert、panic，其实是看懂了sys_printx。

50分钟，大概只有最后20分钟才真正开始看明白，前30分看着代码琢磨，看不明白。后20分钟，我一边断点调试看效果一边理解，很快理解了。

1. `if(!*p)`。
   1. 只有`*p`是空字符时，才为真。
   2. `*p`是空格时，是0。

不理解代码的时候，就断点调试吧。

### 2021-04-30 18:01

断点调试又一次大显神威，让我理解了之前理解不了的msg_send函数。

