# 输入输出系统---printf的实现

## 代码位置

/home/cg/os/pegasus-os/v27

## 回忆

### v1

printf的函数原型：`int printf(char *format, var_list)`。

我只实现`%x`。`%x`是啥？是十进制整数吗？`%d`才是整数，不是吗？

#### vsprintf

原型：`vsprintf(char *buf, char *format, var_list)`。

几个疑问：

1. buf的长度是多少？
2. var_list是参数列表，数据类型是什么？

流程是：

1. 把format逐字符复制到buf，遇到`%`时跳过。
2. 遇到`%`的下一个字符时，用var_list中的数据替换。
3. 指向var_list的指针往后移动4个字节。这种叙述可能不准确。要完成的事是：下次再遇到`%`后的下一个字符时，用var_list中的下一个数据替换。

> 20分钟。

## 学习

### v1

#### strcpy

用汇编实现。

原型：`char* strcpy(char *dest, char *src)`。

##### 回忆

怎么实现？

用到两个寄存器：`edi`、`esi`。

1. `[ds:esi]`。
2. `[es:edi]`。

四个寄存器的搭配是否正确？

怎么把寄存器指向src？

##### 学习

1. 没有写出ds、es寄存器。
2. 直接把edi、esi指向第一个参数、第二个参数。
3. 返回值放在eax中。

#### strlen

用汇编实现。

原型：`int strlen(char *str)`。

把esi指向str，然后逐字节遍历，用esi记录长度。

退出循环的条件是当前字符是空。

#### 打印字符

使用系统调用实现printf，最终执行打印的函数怎么实现？

使用`out_char`完成。

### v2

#### 系统调用write

增加系统调用使用一个模板。

函数原型：`int write(char *buf, int len)`。

在write里面，通过传递参数，让系统调用中断例程选择合适的系统函数完成打印。

进程和tty绑定，write也和tty绑定。

在操作系统中，打印使用out_char完成。

无难度，但我想写成和于上神的的代码一样，又不记得他的代码的所有细节，所以，感觉有困难。

`void out_char(TTY *tty, char key)`。

`void sys_write(char *buf, int len, TTY *tty)`。

只需要这两个参数，就能完成了。于上神的代码，还有一个参数。

#### tty和进程



## 写代码

> 写代码和调试放到一起。

1. 在sys_call_table增加sys_write。
2. 在main.c实现sys_write。
3. 在syscall.asm实现函数write。
4. 在进程表增加tty的index。

> 写起来很顺利。没有理解不了的地方。除了tty和进程的联系。
>
> 耗时2个小时06分。难点：修改系统中断例程；用指针遍历字符串。

### 疑问

一、怎么用指针遍历字符串？

```c
#include <stdio.h>

int main(int argc, char **argv)
{
        char buf[30] = "Hello,World!";
        char *p = buf;

        printf("buf[30] = %s\n", p);

        printf("p = %c\n", *p);
        p++;
        printf("p = %c\n", *p);
        p++;

        printf("p = %c\n", *p);
        printf("p = %s\n", p);

        return 0;
}
```



执行结果：

```shell
[root@localhost c]# gcc -o str str.c -g -m32
[root@localhost c]# ./str
buf[30] = Hello,World!
p = H
p = e
p = l
p = llo,World!
```



二、以前的笔记做得很差

![image-20210425173528546](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210425173528546.png)

没有写清楚不能使用的原因。当时我应该很清楚，但现在完全想不起来了。记忆力非常不靠谱。

### 漏掉了

```c
// 只支持%x
void Printf(char *fmt, ...);
void vsprintf(char *buf, char *fmt, char *var_list);
void write(char *buf, int len);
// printf end
```

#### Printf

`void Printf(char *fmt, ...)`。

#### vsprintf

`int vsprintf(char *buf, char *fmt, char *var_list)`。

> 自己不会写，就去看别人写的代码。
>
> 我未尽全力去想。
>
> 可是，尽全力想出来的又如何？一样会忘记。

##### v1

1. 概况思路：保留fmt中的非%x字符串，把%x替换成可变参数。
2. 只说vsprintf的实现方法。
3. `int vsprintf(char *buf, char *fmt, char *var_list)`。
   1. buf存储处理后的字符串。
   2. fmt是原始字符串。
   3. var_list是用来替换fmt中的%x的数字。
4. 声明 char *p。用p指向buf。
5. 遇到%，跳过。
6. 遇到x，处理。
   1. 用var_list的第一个参数，转为十六进制字符串。
   2. 把十六进制字符串复制到buf。
   3. 再次遇到x时，用var_list的下一个参数。

## 总结

## 调试

```shell
[root@localhost c]# yum debuginfo-install glibc-2.28-127.el8.x86_64
Last metadata expiration check: 1:11:21 ago on Sat Apr 24 17:02:02 2021.
Could not find debuginfo package for the following installed packages: glibc-2.28-127.el8.x86_64
Could not find debugsource package for the following installed packages: glibc-2.28-127.el8.x86_64
Dependencies resolved.
Nothing to do.
Complete!
```



```shell
$20 = 0xffffd314 "\003"
(gdb) p ((int *)(var_list))
$21 = (int *) 0xffffd314
(gdb) p var_list
$22 = 0xffffd314 "\003"
(gdb) p fmt
$23 = 0x80485ac "I am %d years old\n"
(gdb) p &fmt
$24 = (const char **) 0xffffd2e0
(gdb) p (char *)&fmt
$25 = 0xffffd2e0 "\254\205\004\b\024\323\377\377\300\317\377\367"
(gdb) p (char *)&fmt + 4
$26 = 0xffffd2e4 "\024\323\377\377\300\317\377\367"
(gdb) p (char *)((char *)&fmt + 4)
$27 = 0xffffd2e4 "\024\323\377\377\300\317\377\367"
(gdb) p (char *)((char *)&fmt + 4 + 8)
$28 = 0xffffd2ec ""
(gdb) p (char *)((char *)&fmt + 4 + 4)
$29 = 0xffffd2e8 "\300\317\377\367"
(gdb) p (char *)((char *)(&fmt) + 4)
$30 = 0xffffd2e4 "\024\323\377\377\300\317\377\367"
(gdb) p &(char *)((char *)(&fmt) + 4)
Attempt to take address of value not located in memory.
(gdb) p *(char *)((char *)(&fmt) + 4)
$31 = 20 '\024'
(gdb) p (char *)((char *)(&fmt) + 4)
$32 = 0xffffd2e4 "\024\323\377\377\300\317\377\367"
```



重新断点打印，矛盾之处消失了：

```shell
Breakpoint 1, test (fmt=0x80485ac "I am %d years old\n") at var_list.c:17
17		char *var_list = (char *)((char *)(&fmt) + 4);
(gdb) p (char *)((char *)(&fmt) + 4)
$1 = 0xffffd314 "\003"
(gdb) p &fmt
$2 = (char **) 0xffffd310
```

### 函数的可变参数

```shell
#include <stdio.h>

void test(char *fmt, ...);
void test2(const char *fmt, char *var_list);

int main(int argc, char **argv)
{
        char *fmt = "I am %d years old\n";
        //test(fmt, 3, 5);
        //char *var_list = (char *)fmt + 4;
        test(fmt, 3, 5, 4, 8);
        return 0;
}

void test(char *fmt, ...)
{
        char *var_list = (char *)((char *)(&fmt) + 4);
        test2(fmt, var_list);
        return;
}

void test2(const char *fmt, char *var_list)
{
        printf(fmt);
        return;
}
```

下面是用gdb断点调试上面的代码的过程的节选：

```shell
(gdb) p &fmt
$18 = (char **) 0xffffd310
(gdb) p (char *)&fmt
$19 = 0xffffd310 "\254\205\004\b\003"
(gdb) p fmt
$20 = 0x80485ac "I am %d years old\n"
(gdb) x /cw 0x80485ac
0x80485ac:	73 'I'
(gdb) x /xw 0xffffd310
0xffffd310:	0x080485ac

(gdb) p fmt
$20 = 0x80485ac "I am %d years old\n"
(gdb) x /cw 0x80485ac
0x80485ac:	73 'I'
(gdb) x /xw 0xffffd310
0xffffd310:	0x080485ac
(gdb) x /cw 0x80485ac
0x80485ac:	73 'I'
(gdb) x /cw0xQuit85ac + 1)
(gdb) x /cw (0x80485ac +1)
0x80485ad:	32 ' '
(gdb) x /cw (0x80485ac +2)
0x80485ae:	97 'a'
(gdb) x /cw (0x80485ac +3)
0x80485af:	109 'm'
(gdb) x /cw (0x80485ac +4)
0x80485b0:	32 ' '
(gdb) x /cw (0x80485ac +5)
0x80485b1:	37 '%'
(gdb) x /cw (0x80485ac +6)
0x80485b2:	100 'd'

(gdb) p 0x80485ac
$21 = 134514092
(gdb) p /x 0x80485ac
$22 = 0x80485ac

(gdb) p &fmt
$23 = (char **) 0xffffd310
(gdb) p *&fmt
$24 = 0x80485ac "I am %d years old\n"
```

1. `&fmt`是获取fmt的内存地址。
2. `(char *)&fmt`，并不会改变fmt的内存地址，相当于数据类型转换，意思是，&fmt的内存地址指向的那块内存包含1个字节。
3. fmt是字符串`"I am %d years old\n"`的首字母的内存地址。
4. `char *fmt = "I am %d years old\n";`，怎么理解这句？
   1. `I am %d years old\n`存储在一片内存中。
   2. 这片内存的第一个字节的地址是：`I`的内存地址。
   3. `char *fmt`，把`I`的内存地址存储在另外一个字节中。
   4. 这个字节的地址是：&fmt。
5. 更正：
   1. 第一点的说法：`&fmt`是获取fmt的内存地址。
   2. 不知道怎么更正。平时，大家都是这么说的。
   3. &fmt代表一块内存，这块内存中存储的是另外一块内存的地址，那个地址的内存中存储的是字符串的第一个字符。

> printf的难点和时间消耗点是理解可变参数。
>
> 可变参数和堆栈的关系，不难理解。
>
> 难理解的是，gdb断点过程中的打印数据，与我的理解矛盾。

#### 难点

```shell
Breakpoint 2, test (fmt=0x80485ac "I am %d years old\n") at var_list.c:17
17		char *var_list = (char *)((char *)(&fmt) + 4);
(gdb) p var_list
$25 = 0x0
(gdb) s
18		test2(fmt, var_list);
(gdb) p var_list
$26 = 0xffffd314 "\003"
(gdb) p &fmt
$27 = (char **) 0xffffd310
```

这仍然是上面的代码的断点过程。

断点输出的第1行，`fmt=0x80485ac`，与第8行`$26 = 0xffffd314`，二个内存地址差别很大，不是加4或减4能计算出来的。

这就是困扰我的地方。

`fmt=0x80485ac`是字符串的第一个字符的内存地址。在函数的堆栈中，存储的是指针类型数据，这个数据是字符串的第一个字符的内存地址。但是，堆栈中存储的并不是第一个字符的内存地址，而是另一个内存地址，在这个内存地址的内存中存储第一个字符的内存地址。

> 难以表达，越说越糊涂。

```shell
Breakpoint 3, main (argc=2, argv=0xffffd3f4) at var_list.c:8
8		char *fmt = "I am %d years old\n";
(gdb) s
11		test(fmt, 3, 5, 4, 8);
(gdb) p &fmt
$37 = (char **) 0xffffd33c
(gdb) s

Breakpoint 2, test (fmt=0x80485ac "I am %d years old\n") at var_list.c:17
17		char *var_list = (char *)((char *)(&fmt) + 4);
(gdb) p &fmt
$38 = (char **) 0xffffd310

(gdb) x /xw 0xffffd33c
0xffffd33c:	0x080485ac
(gdb) x /wx 0xffffd310
0xffffd310:	0x080485ac
```

在main函数中、test函数中，fmt的内存地址不一样。

fmt的数据类型是指针，这意味着，它的值是一个内存地址。

&fmt是求fmt的内存地址还是求fmt的值？

二者是不同的。

fmt的数据类型是指针，这意味着：

1. 四个字节的内存M。
2. M的第一个字节的地址。
3. M内的数据A。
4. A这个地址的内存中的数据。

&fmt，是上面四者中的哪个？是第2个。

进入函数test后，&fmt获取的是M的地址。

M是什么？M是test的堆栈中的四个字节。根据C函数调用规约，

1. test的堆栈中此时存储它的参数、调用test的指令的下一条指令的地址等。
2. M存储最后一个参数的内存地址。
3. 用M作为参照系，test的第二个、第三个和其他参数的内存地址分别存储在 &fmt + 4、&fmt + 8。
4. 更正第3点。第二个、第三个参数的内存地址分别存储在`(char *)&fmt + 4`、`(char *)&fmt + 8`。
5. 为什么这样更正？
   1. 在32位模式下，栈的每个元素占用4个字节，因此 4、8的单位必须是字节。而4、8的单位由(char *)决定。
   2. (int *)&fmt + 4，结果是在&fmt的值上加 4 * 4个字节。
   3. (char *)&fmt + 4，结果是在&fmt的值上加4 * 1个字节。

```shell
(gdb) p &fmt
$39 = (char **) 0xffffd310
(gdb) p &fmt+4
$40 = (char **) 0xffffd320
(gdb) p (int *)&fmt + 4
$41 = (int *) 0xffffd320
(gdb) p (short *)&fmt + 4
$42 = (short *) 0xffffd318
(gdb) p (int *)&fmt + 4
$43 = (int *) 0xffffd320
(gdb) p (long *)&fmt + 4
$44 = (long *) 0xffffd320
(gdb) p (long long *)&fmt + 4
$45 = (long long *) 0xffffd330
(gdb) p &fmt
$46 = (char **) 0xffffd310
```

上面的执行结果证明了：`(char *)&fmt + 4`中的`(char *)`决定了4的单位是字节。

至此，对指针的理解，又增加了新知识。对于printf的可变参数，不再有疑问。

简单地说，可变参数是怎么实现的？

1. 根据第一个参数，获取函数的第一个参数的地址。
2. 这个地址，是该参数的堆栈中的第一个元素。
3. 根据C语言中的函数调用规约，该参数的其他参数都在它的堆栈中，而且，与第一个参数（堆栈的第N个元素）相邻，内存地址递增，增幅是4个字节（32个bit)。
4. `(char *)`有两个作用：
   1. 规定`(char *)&fmt + 4`中的4的单位是字节。
   2. `(char *)&fmt`规定，从&fmt这个内存地址指向的那块内存开始，只有一个字节（包括&fmt指向的那个字节）存储的数据是(char *)&fmt指向的数据。
      1. 更简单的说法，(char *)规定存储数据的内存只有一个字节。
      2. (int *)规定存储数据的内存有4个字节。

> 耗费6个小时，理解printf的可变参数。
>
> 两大主要难点：
>
> 1. 可变参数的实现原理。简单，昨晚大概就看明白了。
> 2. 根据第一个参数获取其他参数的内存地址。这是最大难点。更具体地说，理解那些C语言语法。所以，难点是C的指针语法。



### gdb故障

问题：

```html
Missing separate debuginfos, use: debuginfo-install glibc-2.12-1.47.el6_2.9.i686 libgcc-4.4.6-3.el6.i686 libstdc++-4.4.6-3.el6.i686
```

解法方法：

```shell
# debuginfo-install is a command of yum-utils, so
yum install yum-utils
debuginfo-install glibc
# if the warning's still there, edit /etc/yum.repos.d/CentOS-Debuginfo.repo, set enabled=1
```

### 终极调试

#### 弄错了代码

1. 编译的是v27，执行的却是v26-1。

> 耗费时间20分钟。

#### 不正常

1. 不能切换终端
2. 不知道当前是哪个终端
3. 不能在当前终端输入字符

##### 怎么调试

一、总是执行TTY进程时，正常。

是进程调度出了问题吗？也就是说，其他进程例如TestA、TestB等会被执行吗？

怎么验证？

在TestA中设置断点。

TestA被执行了。

二、Printf是否出了问题？

1. 注释使用了Printf的语句，看看结果。-----> 正常。能确定，Printf有问题。
2. 怎么测试Printf？
   1. 在无进程切换的场景下测试。
   2. 能确定，是Printf出了问题。

三、单独测试并修改Printf

1. 脱离进程测试它。

> 有进展。Printf函数有问题，复制字符串有错误。
>
> 怎么发现的？
>
> 1. 在单进程发现了有问题，但不知道怎么解决。
> 2. 对比于上神的代码，立刻知道了，我的代码为啥是错的。



> 耗费了2个小时10分。

四、Printf中的write成功触发了系统调用中断吗？

1. 使用bochs在write中设置断点。

```assembly
; ====================================================================================
;                                   save
; ====================================================================================
save:
        pushad          ; `.
        push    ds      ;  |
        push    es      ;  | 保存原寄存器值
        push    fs      ;  |
        push    gs      ; /
        mov     dx, ss
        mov     ds, dx
        mov     es, dx

        mov     esi, esp                    ;esi = 进程表起始地址

        inc     dword [k_reenter]           ;k_reenter++;
        cmp     dword [k_reenter], 0        ;if(k_reenter ==0)
        jne     .1                          ;{
        mov     esp, StackTop               ;  mov esp, StackTop <--切换到内核栈
        push    restart                     ;  push restart
        jmp     [esi + RETADR - P_STACKBASE];  return;
.1:                                         ;} else { 已经在内核栈，不需要再切换
        push    restart_reenter             ;  push restart_reenter
        jmp     [esi + RETADR - P_STACKBASE];  return;
                                            ;}


; ====================================================================================
;                                 sys_call
; ====================================================================================
sys_call:
        call    save
	push	dword [p_proc_ready]
        sti

	push	ecx
	push	ebx
        call    [sys_call_table + eax * 4]
	add	esp, 4 * 3

        mov     [esi + EAXREG - P_STACKBASE], eax
        cli
        ret


```



上面是于上神的系统调用中断例程。我有疑惑：

1. esi并没有入栈，经历了call语句后，能保证esi不被修改吗？
2. 给call调用的函数增加了三个参数，如果有的函数只有2个参数或者有4个参数，怎么满足要求？
   1. 调试证明，对于有3个参数的函数，入栈4个数据，会导致传参错误。

> 耗费了1个小时28分，一直在调试。

###### 小结

一直在调试，过程如下：

1. Kernel.asm中，调用sys_write时，传参不正确。
   1. 怎么发现的？使用bochs的断点，能一眼看出ebx、ecx是否正确，看不出proc_ready_table是否正确。
   2. 使用bocsh + gdb调试，在sys_write中，看到第3个参数不正确。
2. 修改正确了：sys_write只有三个参数，就按顺序只入栈3个数据。暂时不管esi被修改的问题。
3. 又遇到奇怪的问题：bochs出现死循环，写数据超出限制。
4. 使用bochs + gdb 调试，发现 proc_ready_table 一直正常，直到遇到sys_write断点。
5. 在TestB中使用Printf，再查看sys_write断点。
   1. 参数正常。
   2. 不能切换终端。

##### v1

新发现：

1. 在TestA使用Printf会导致错误。
2. 使用get_ticks会导致错误。

> 耗费时间1小时58分。
>
> 在做什么？
>
> 小修改，然后运行调试。
>
> 唯一的成果，就是确定了上面两点。
>
> 多进程调试，好困难。根据断点调试的数据，也无法确定代码的执行流程。

不是get_ticks等系统调用的问题。

##### v2

TestA从restart中启动，是第一个进程。此时，TTY还没有初始化。

不是系统调用出了问题，而是，在TTY初始化前使用了TTY。

修改：第一个启动的进程是，TTY。

没有用，错误仍然出现。

```shell
00014033826i[BIOS  ] Booting from 0000:7c00
00022846750e[CPU0  ] LLDT: selector.ti != 0
00023109011e[CPU0  ] write_virtual_checks(): write beyond limit, r/w
```

在TestA中使用`disp_str_colour("A", 0x0A);`能打印出字符串。

使用`Printf("a:%x", 1);`就会出错。

![image-20210426213909111](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210426213909111.png)



上图中画红线的信息，无用。与代码中的len毫无关系。

又发现了奇怪的线索。

```shell
(gdb) p proc_ready_table
$20 = (Proc *) 0x0
```

这是断点打印TestA的截取数据。

执行完TestA中的打印语句后，我发现当前进程的内存地址是0。

再验证一次，是否，进入TestA的时候，当前进程就已经变为0。

刚进入TestA的时候，当前进程正常。

检查：第一次执行完write，当前进程是否正常。

第一次执行完write后，当前进程的内存地址就已经变成了0x0。我是通过在在调用write函数的后面加return，然后在return设置断点观察到的。见下图：

![image-20210426220239363](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210426220239363.png)



![image-20210426220308665](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210426220308665.png)



我认为，write系统调用在TestA中出了问题。

很奇怪，为什么唯独在TestA中出问题？

当前，代码的执行流程：

1. TaskTTY
2. TestA
   1. write
   2. sys_call
   3. 恢复TestA，proc_ready_table出了问题。

报错：

```shell
00022571641i[CPU0  ] | EAX=000053f0  EBX=00000006  ECX=00ff53f6  EDX=000003d5
00022571641i[CPU0  ] | ESP=fffffff0  EBP=0003544c  ESI=00038c60  EDI=00035341
00022571641i[CPU0  ] | IOPL=1 id vip vif ac vm rf nt of df if tf sf zf AF pf cf
00022571641i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
00022571641i[CPU0  ] |  CS:0008( 0001| 0|  0) 00000000 ffffffff 1 1
00022571641i[CPU0  ] |  DS:0030( 0006| 0|  0) 00000000 ffffffff 1 1
00022571641i[CPU0  ] |  SS:0030( 0006| 0|  0) 00000000 ffffffff 1 1
00022571641i[CPU0  ] |  ES:0030( 0006| 0|  0) 00000000 ffffffff 1 1
00022571641i[CPU0  ] |  FS:000d( 0001| 1|  1) 00000000 ffffffff 1 1
00022571641i[CPU0  ] |  GS:0039( 0007| 0|  1) 000b8000 0000ffff 0 0
00022571641i[CPU0  ] | EIP=0003053d (000305ed)
00022571641i[CPU0  ] | CR0=0x60000011 CR2=0x00000000
00022571641i[CPU0  ] | CR3=0x00000000 CR4=0x00000000
00022571641i[CPU0  ] 0x000305ed>> lldt word ptr ss:[esp+68] : 0F00542444
00022571641i[CMOS  ] Last time is 1619446560 (Mon Apr 26 07:16:00 2021)
00022571641i[XGUI  ] Exit
00022571641i[      ] restoring default signal behavior
```

这个线索毫无用处。

###### Memset

非常非常怪异的数据：

```shell
Breakpoint 1, vsprintf (buf=0x3533c <proc_stack+188> "", fmt=0x321fd "a:%x", var_list=0x35458 <proc_stack+472> "\001") at main.c:1571
1571	{
(gdb) p *proc_ready_table
$1 = {s_reg = {gs = 57, fs = 13, es = 13, ds = 13, edi = 3296950509, esi = 10598404, ebp = 2024663120, kernel_esp = 4263789636,
    ebx = 3092309040, edx = 3897557153, ecx = 3296950453, eax = 4282812932, eip = 201020, cs = 5, eflags = 4610, esp = 218240, ss = 13},
  ldt_selector = 72, ldts = {{seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 187 '\273',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
      seg_base_middle = 0 '\000', seg_attr1 = 179 '\263', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 0,
  ticks = 5, priority = 5, tty_index = 1}
(gdb) s
1577		Memset(tmp, 0, sizeof(tmp));
(gdb) p *proc_ready_table
$2 = {s_reg = {gs = 57, fs = 13, es = 13, ds = 13, edi = 3296950509, esi = 10598404, ebp = 2024663120, kernel_esp = 4263789636,
    ebx = 3092309040, edx = 3897557153, ecx = 3296950453, eax = 4282812932, eip = 201020, cs = 5, eflags = 4610, esp = 218240, ss = 13},
  ldt_selector = 72, ldts = {{seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 187 '\273',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
      seg_base_middle = 0 '\000', seg_attr1 = 179 '\263', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 0,
  ticks = 5, priority = 5, tty_index = 1}
(gdb) s
1578		char *next_arg = var_list;
(gdb) p *proc_ready_table
$3 = {s_reg = {gs = 4026531967, fs = 4026597203, es = 4026597203, ds = 4026597203, edi = 4026597203, esi = 4026597203, ebp = 4026597203,
    kernel_esp = 4026597203, ebx = 4026597029, edx = 4026591623, ecx = 4026591718, eax = 4026591718, eip = 4026591718, cs = 4026591718,
    eflags = 4026593111, esp = 4026591718, ss = 3221225814}, ldt_selector = 63565, ldts = {{seg_limit_below = 61440,
      seg_base_below = 63553, seg_base_middle = 0 '\000', seg_attr1 = 240 '\360', seg_limit_high_and_attr2 = 254 '\376',
      seg_base_high = 227 '\343'}, {seg_limit_below = 61440, seg_base_below = 59193, seg_base_middle = 0 '\000', seg_attr1 = 240 '\360',
      seg_limit_high_and_attr2 = 89 'Y', seg_base_high = 248 '\370'}}, pid = 4026591278, ticks = 4026593234, priority = 4026568565,
  tty_index = 4026590962}
(gdb)
```

执行完这句`1577		Memset(tmp, 0, sizeof(tmp));`后，proc_ready_table就不正常了。

我把 `1577		Memset(tmp, 0, sizeof(tmp));`注释掉试试。

终于找到了问题，就是 `Memset(tmp, 0, sizeof(tmp))`导致错误。它修改了esi。

> 耗费时间1小时30分。

![image-20210426230458156](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210426230458156.png)

为什么执行`Memset(tmp, 0, sizeof(tmp))`会让proc_ready_table的数据错误？

分别把Memset的第三个参数设置为256、128、64，或者干脆注释掉这句，我的结论是：

这句正好把proc_ready_table指向的那片内存的当前进程数据擦除了。

这就奇怪了，难道不能在程序中随便使用Memset等函数吗？

与进程堆栈和内核堆栈大小都没有关系。

我猜测：和整体内存布局有关。

```shell
vsprintf (buf=0x134b3c <proc_stack+188> "", fmt=0x321fd "a:%x", var_list=0x134c58 <proc_stack+472> "\001")

(gdb) s
1583		Memset(tmp, 0, sizeof(tmp));
(gdb) p &tmp
$1 = (char (*)[256]) 0x134a08 <gdt+936>

(gdb) p &proc_stack
$3 = (int (*)[13312]) 0x134a80 <proc_stack>

(gdb) p proc_ready_table
$6 = (Proc *) 0x0
(gdb) p &proc_ready_table
$7 = (Proc **) 0x134a60 <proc_ready_table>


```

将tmp的大小改成64后，断点数据如下：

```shell
Breakpoint 1, vsprintf (buf=0x134b3c <proc_stack+188> "", fmt=0x321fd "a:%x", var_list=0x134c58 <proc_stack+472> "\001") at main.c:1575
1575	{
(gdb) s
1583		Memset(tmp, 0, sizeof(tmp));
(gdb) p &tmp
$1 = (char (*)[64]) 0x134ac8 <proc_stack+72>

(gdb) p &gdt
$3 = (Descriptor (*)[128]) 0x134660 <gdt>
```

1. `(char (*)[256]) 0x134a08 <gdt+936>`
2. `(char (*)[64]) 0x134ac8 <proc_stack+72>`
3. 第1条数据，`<gdt+936>`是表示已经进入gdt中了吗？而第二条数据，表示仍然在进程的堆栈中。

暂时先这样吧。

不过，真的是一个相当神奇的问题。进程中的变量，而且是函数中的变量，怎么会擦除全局的、操作系统变量呢？

一个问题：Memset中的tmp[256]究竟是破坏了gdt还是擦除了全局的proc_ready_table？gdt被破坏，很多东西都会乱套。

> 耗费1小时2分。
>
> 发现了奇怪的问题，未能从根本从解决，但通过修改数据暂时解决了问题。

## 大难题

在函数中创建过大的局部变量，为啥会破坏全局变量gdt中的数据？

问题场景见：v2#Memset。