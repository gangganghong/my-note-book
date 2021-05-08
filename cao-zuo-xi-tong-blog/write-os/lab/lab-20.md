# 实现IPC

## 代码位置

/home/cg/os/pegasus-os/v29

## 学习

### v1

#### 调试函数

##### sys_printx

这个函数被汇编代码写的汇编函数sys_call调用。

printx调用write，write触发sys_call调用sys_printx。printx是实现调试函数的基础。

在sys_printx中，执行printx时CPU的特权级是0或者调用方是panic，打印错误信息，然后关闭中断、hlt。关闭中断，让CPU在因执行hlt而停止之前不切换到其他代码。执行printx时CPU的特权级是1~3并且调用方不是panic，打印错误信息，然后正常返回。

重点不是打印错误信息。想简化工作，可以只打印错误信息就行。重点是，用户进程，会让CPU继续执行其他指令；内核代码，会让CPU停止。

执行printx时CPU的特权级和在sys_printx中k_reenter的关系是：

1. k_reenter大于0时，特权级是0。
2. k_reenter等于0时，特权级是1~3。

> 时间久了，这个规则，我又会忘记。
>
> 这种判断方法，非常不好。严格来说，特权级的值是k_reenter的值的充分条件，非充要条件（无法证明）。

spin

打印错误信息，然后使用死循环让CPU停止执行其他代码。

内联汇编

语法：`__asm__("ud2")__`。

##### panic

打印错误信息，让操作系统hlt。

##### assert

打印错误信息，根据特权级决定让CPU死循环还是hlt。

> 我独立写出了上面的知识，却是在背诵。

#### IPC函数

不同进程之间传递数据，这就是IPC。

怎么做到？有点混乱。

##### send

###### 简单流程

A进程中有变量var1，B进程中有变量var2。要把var1的值复制给var2。

过程是这样的：

1. B进程先从A接收数据。由于A还没有向它发送数据，因此B的状态变为RECEIVING。
2. A进程运行。把var1数据复制给var2。
   1. 在A进程中不能做这个操作。A进程不能访问var2，不能访问B进程的任何数据。
   2. A进程通过send完成这个操作。执行send时，CPU的特权级是0。
   3. 在send中，var1通过参数传递进来。var2怎么进入send？
   4. 在A中调用send，不能直接使用var2，能直接使用B的进程ID。
   5. 根据B的进程ID，能计算出B的进程表。
   6. B的进程表包含var2的内存地址吗？
   7. B调用receive时，在receive中，可以把var2的内存地址（在进程B中的内存地址）记录到进程表中。
   8. var1的线性地址 = A进程的数据段的基地址 + var1的偏移量，var2的线性地址 = B进程的数据段的基地址 + var2的偏移量。
   9. 没有开启分页机制前，线性地址等于物理内存地址。
   10. 在0特权级下，CPU能访问任何地址。把var1的数据复制到var2。

###### 进程的运行，有顺序要求吗？

就算有顺序要求，也不过分。进程的运行，总会有先有后。例如，服务端先运行，客户端后运行。

###### 完整流程

A进程向B进程发送消息。

1. A进程向B发送消息，B是RECEIVE状态。
   1. 把消息复制给B。
   2. 解除B的阻塞。
   3. A进程继续执行。
2. A进程向B发送消息，B不是RECEIVE状态。
   1. A进程转变为SENDING状态。
   2. 把A进程放入B进程的待发送队列。

> 写得这么简单，漏掉了很多内容吗？

##### receive

###### 完整流程

A进程向B进程发送消息，也就是，B进程从A进程接收消息。

1. B进程从A进程接收消息，A进程并没有向B发送消息。
   1. A进程并没有向B发送消息。
      1. A进程的状态不是SENDING。
      2. A进程发送的消息的目标不是B进程。
   2. B进程的待接收队列不是空。
      1. 在接收队列中查询A进程
         1. 找到了，
            1. 把消息从A进程复制到B进程。
            2. 解除A进程的阻塞。
            3. 把A进程从B的待接收队列中移除。
            4. 继续执行B进程。
         2. 没找到，阻塞B进程。
            1. 应该阻塞还是继续执行？
            2. 应该阻塞。因为，我设计的IPC是同步的。没有获取到数据继续执行，会导致错误。
2. B进程从A进程接收消息，A进程向B发送消息。
   1. A进程向B发送消息。
      1. A进程的状态是SENDING。
      2. A进程发送的消息的目标是B进程。
   2. 把A进程的消息复制到B进程。
   3. B进程继续执行。

B进程还能接收任意来源的消息。流程如下：

1. B进程接收任意来源的消息。
2. 待接收队列不是空，
   1. 选取接收队列的第一个进程F，并且把它从接收队列中移除。
   2. 把F的消息复制到B，解除F的阻塞。
   3. B继续执行。
3. 待接收队列是空，阻塞B，B的状态转变为RECEIVING。

##### 死锁

A向B发送消息，B向A发送消息，这就是死锁。

在哪里检查死锁？发送？接收？还是二者都需要检查死锁？

在发送中，会出现死锁吗？会。

发送和接收是一对。不成对出现，就会一直阻塞。

《一个操作系统的实现》中的死锁处理并不是重点，也许不全面。我不在这个知识点上纠结太多时间。甚至，我可以等时间充足了再实现它。

##### 疑问

一、怎么计算进程内的变量的绝对地址？

LDT[ds的索引部分] + 变量名。

上面的公式不正确。

应该是：数据段的基地址 + 偏移量。

数据段的基地址：从LDT[ds的索引部分]中获取。LDT[ds的索引部分]的值是数据段的描述符。

偏移量：用变量名表示。

二、使用全局变量传递消息，能行吗？

在用户进程中，能访问全局变量。如此，“保护”二字，体现在哪里？

文件socket和操作系统中的全局变量有什么关系？

## 写代码

### 进程表修改

#### 新增成员

在进程表增加IPC要使用的成员：

1. int p_flags，进程的状态：
   1. RUNNING：0。
   2. RECEIVING：1。
   3. SENDING：2。
2. Message *p_msg，消息体。
3. int p_send_to，要发消息给谁，目标进程的ID。
4. int p_receive_from，要从哪个进程接收消息，目标进程的ID。
5. MsgSender p_send_queue，要给本进程发送消息的进程的队列。
6. char p_name[20]，进程名称。

#### 发送消息的进程队列

队列中的元素是一个结构体。

```c
typedef struct{
  	Proc *sender;
  	Proc *next;
}MsgSender;
```



### 调试函数

#### write_debug

通过sys_call调用sys_printx。

原来的sys_write不改动。

#### sys_printx

函数原型：`void sys_printx(char *error_msg, int caller_pid)`。

简化这个函数，无论调用方是谁，都用相同的方式打印字符串。

这是在内核中，使用out_char还是disp_str打印字符串？

使用out_char，因为，现在的打印，和TTY关联起来了。

计算出error_msg的线性地址（等同物理地址），然后逐字符打印。详细流程如下：

1. 计算error_msg的线性地址。
   1. 当k_reenter = 0，调用方的特权级是1~3。线性地址 = 调用方进程的数据段的基地址 + error_msg。
   2. 当k_reenter > 0，调用方的特权级是0。线性地址 = 数据段的基地址 + error_msg。
2. 逐字符打印error_msg。

#### vsprintf

增加对%c、%s、%d的支持。

##### %c

1. char str[2] = {char, "\0"};
2. Strcpy(p, str)；

##### %s

1. 使用`*((char **)next_arg_list)`获取字符串。
2. 使用Strcpy连接到p。

##### %d

1. 把整型数据转成字符串形式。例如，把355转成字符串"355"。
2. 使用Strcpy连接到p。

怎么把355转成字符串"355"？


$$
355 = 3 * 100 + 5*10 + 5*1
$$
我的思路：

1. 声明一个数组arr，数组大小是无符号整型数的最大值。这已经远远超出使用需求了。
2. 355除以10，把余数放到数组arr中。
3. 反复执行第2步，一直到商是0时停止。当商是0时，记录下当前数组索引n。
4. 再申明一个和arr相似的数组，差别是这个数组的名字是arr2。
5. 从后往前把数组arr的元素复制到数组arr2
6. `arr2[index+1] = '\0'`。

> 这种方法，太浪费空间。时间复杂度是O(N)。
>
> 于上神的代码使用了嵌套指针，理解不了。

> 有点难。

于上神的代码：

```c
char *i2a(int val, int base, char **ps)
{
  	int remainer = val % base;
  	int quotient = val / base;
  	if(quotient > 0){
    		i2a(quotient, base, ps);	
    }
  	
  	*(*ps)++ = remainer + '0';
  
  	return *ps;
}
```

使用了递归，比较难理解。也不用去理解。递归是正确的。递归的写法是：

1. 没有递归的时候，处理逻辑。
2. 满足什么条件，进入递归。

#### printx

函数原型：`void printx(const char *fmt,...)`。

系统调用，在调试函数中使用。

不修改已经存在的Printf。

#### spin

函数原型：`void spin(char *error_msg)`。

1. ~~使用printx打印错误信息。~~
2. 死循环。
3. 打印错误信息交给sys_printx，spin只执行死循环。

#### panic

函数原型：`void panic(char *error_msg)`。

1. 在错误信息的开头加入魔数，PANIC_MAGIC，58。
2. 使用printx打印错误消息。

#### assert

一个宏。

逻辑流程：

1. 表达式是true，什么也不做。
2. 表达式是false，执行assertion_failure。

代码是：

```c
#define assert	if(exp)
	else	assertion_failure(#exp, __FILE__, __BASE_FILE__, __LINE__)
```

#### assertion_failure

函数原型：`void assertion_failure(char *exp, char *filename, char *base_filename, unsigned int line)`。

1. 在错误字符串的前面加上魔数，ASSERT_MAGIC，值是59。
2. 这个很随意。我把魔数设计成一个字符，方便在sys_printx中解析。弊端是，把与魔数相同的字符识别为了魔数。
3. 使用printx打印错误信息。
4. 使用spin打印错误信息。第3步执行后CPU没有停止才执行到这一步。

### IPC

操作系统必须提供接收消息、发送消息两种功能。

使用一个系统调用和两个系统调用都行。

我选择用两个系统调用实现。没有特别理由，都行。

#### sys_send_msg

`int sys_send_msg(Proc *sender, Proc *receiver, Message *msg)`。

流程：

1. receiver的状态是RECEIVING，而且，receiver的消息来源是sender或ANY。
   1. 从sender中把消息复制到receiver。
   2. 重置sender。
      1. p_msg = 0
      2. p_send_to = 0
   3. 重置receiver。
      1. p_flags = 0
      2. p_msg = 0
      3. p_receive_from = 0
   4. 解除receiver的阻塞。
2. 不满足上面的条件。
   1. 把sender中的消息放入receiver的p_send_queue。
   2. 阻塞sender。

被sys_call调用。

#### sys_receive_msg

`int sys_receive_msg(Proc *receiver, Proc *sender, Message *msg)`。

流程如下：

1. sender是ANY。
   1. 检查p_send_queue。
      1. 为空，copyOK = 0。
      2. 不为空，获取队列的第一个进程，copyOk = 1。p_from是第一个进程。
2. sender是某个特定进程。
   1. sender 是SENDING状态，并且，sender的p_send_to是ANY或本进程。
      1. 把sender从p_send_queue中移除。
      2. copyOK = 1。
      3. p_from是sender。
   2. 不满足上面的条件。
      1. ~~把receiver的状态修改为RECEIVING。~~
      2. ~~阻塞receiver。~~
3. copyOK = 1
   1. 把消息从p_from复制到receiver。
   2. 重置p_from。
   3. 重置receiver。
4. copyOK = 0
   1. 把receiver的状态修改为RECEIVING。
   2. src = ANY。把receiver的p_receiver_from设置成ANY。
   3. src = 特定进程。把receiver的p_receiver_from设置成特定进程。
   4. receiver的p_msg设置成m。
      1. 不理解。
   5. 阻塞receiver。

被sys_call调用。

#### send_msg

系统调用。

#### receive_msg

系统调用。

#### send_rec

对send_msg、receive_msg进行封装。很有必要。

> 理不清头绪。
>
> 不必受于上神的代码的束缚，我可以根据需求设计一切。

### 用IPC改写get_ticks

### TaskSys

系统任务。运行时，CPU的特权级是1。

### get_ticks

一个系统调用。

### sys_get_ticks

## 难点

### 双指针和字符串

```shell
kernel/vsprintf.c:32:10: error: lvalue required as unary '&' operand
  *ps++ = &((m < 10) ? (m + '0') : (m - 10 + 'A'));
```

```c
PRIVATE char* i2a(int val, int base, char ** ps)
{
	int m = val % base;
	int q = val / base;
	if (q) {
		i2a(q, base, ps);
	}
	*(*ps)++ = (m < 10) ? (m + '0') : (m - 10 + 'A');
	// 等价于
	// unsigned char c = (m < 10) ? (m + '0') : (m - 10 + 'A');
	// *ps++ = &c;

	return *ps;
}
```

`*(*ps)++ = (m < 10) ? (m + '0') : (m - 10 + 'A');`等价于下面的代码：

```c
// 等价于
unsigned char c = (m < 10) ? (m + '0') : (m - 10 + 'A');
*ps++ = &c;
```

已经运行代码测试过了。

怎么理解？

`char ** ps`，

1. ps是一个指向指针的指针。
2. `*ps`仍然是一个指针。指针的值，应该是内存地址，所以，`*ps = &c`。
3. `*(*ps)++ = c`，怎么理解？
   1. int *a;
   2. `*a = 2;`。a的数据类型是指针，`*指针类型的变量`的值应该是指针类型中的那个“类型”的数据。
   3. 上面的两句，是成立的。
   4. `*(*ps)`中的`*ps`是一个指针，`*(*ps)`能够抽象为`*指针类型的变量`。所以，`*(*ps)`的值应该是指针类型中的那个类型，即`char`类型的数据。

```c
char *str = "hello";
char **ps = &str;
```


下面的表格是上面的两个变量的内存示意图


| 内存地址     | 0x06 | 0x07 | 0x08 |      |      |      |      |      |
| ------------ | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 变量名       | ps   | str  |      |      |      |      |      |      |
| 内存中的数据 | 0x07 | 0x08 | h    | e    | l    | l    | o    | \0   |

断点打印出来的数据如下：

```shell
Breakpoint 1, main (argc=2, argv=0xffffd404) at point.c:18
18		char *str = "hello";
(gdb) s
19		char **ps = &str;
(gdb) s
21		return 0;
(gdb) p str
$1 = 0x80485fd "hello"
(gdb) p ps
$2 = (char **) 0xffffd338
(gdb) p &str
$3 = (char **) 0xffffd338
(gdb) x /1wx 0xffffd338
0xffffd338:	0x080485fd
(gdb) p *str
$4 = 104 'h'
(gdb) p *ps
$5 = 0x80485fd "hello"
(gdb) p **ps
$6 = 104 'h'
(gdb) ptype *ps
type = char *
(gdb) ptype **ps
type = char
(gdb) ptype ps
type = char **
```

挺麻烦的。

1. `char *str = "hello"`
2. 看代码：

```c
char *str = "hello";
char *ps = str;
ps = str;
char **ptr = &str;
ptr = &str;
```



### 双指针

`strcpy(q, (*((char**)p_next_arg)));`中的`(*((char**)p_next_arg))`能获取字符。

通过一段简单的代码理解了双指针。把这类问题叫做“双指针”可能不恰当。

简单代码是：

```c
#include <stdio.h>

int main(int argc, char **argv)
{
        //int *a;
        int b = 7;
        int *a = (int *)(&b);
        int c = 9;
        *a = c;
        printf("a = %p\n", a);
        printf("*a = %d\n", *a);
        printf("b = %d\n", b);
        int *e = &b;
        printf("e = %d\n", *e);
        //int *f = b;
        //printf("f = %d\n", *f);

        char *str = "hello";

        return 0;
}
```

设置了两个断点，分别是：`b main`、`b 18`。

断点过程是：

```shell
Breakpoint 2, main (argc=2, argv=0xffffd404) at point.c:18
18		char *str = "hello";
(gdb) s
20		return 0;
(gdb) p str
$1 = 0x80485fd "hello"
(gdb) p &str
$2 = (char **) 0xffffd340
(gdb) x /1wx 0xffffd340
0xffffd340:	0x080485fd
(gdb) p *(char **) 0xffffd340
$3 = 0x80485fd "hello"
(gdb) p *&str
$4 = 0x80485fd "hello"
(gdb) p b
$5 = 9
(gdb) p &b
$6 = (int *) 0xffffd33c
(gdb) p a
$7 = (int *) 0xffffd33c
(gdb) p &a
$8 = (int **) 0xffffd34c
(gdb) p &&a
A syntax error in expression, near `&&a'.
(gdb) p &(&a)
Attempt to take address of value not located in memory.
```

我的分析：

1. `char *str = "hello";`，str的数据类型本来就是`char *`。
2. `&str`的数据类型是什么？`p &str`打印出来的结果是`(char **) 0xffffd340`。
3. 从结果看出，`&str`的数据类型是`char **`。
4. 能总结出什么规则？
5. 内存地址，形式上是一个无符号整型数，数据类型却是一个指针，指针的数据类型是`普通数据类型 *`，例如，`int *`，`char *`。
6. 一个变量的值是内存地址，那么，这个变量的数据类型是一个指针，即`普通数据类型 *`。
7. 如果这个变量的值是一个内存地址，在这块内存地址中的数据，又是一个内存地址，那么，这个变量的数据类型是一个指向指针的指针，即`普通数据类型 **`。
8. 非常明显，指向指针的指针的数据类型是`普通数据类型 **`。
9. `int b`，
   1. `b`的数据类型是`int`。
   2. `&b`是`b`这个变量的内存地址。内存地址的数据类型是指针。`&b`的数据类型是`int *b`。
10. `int *a = (int *)(&b);`
    1. a是一个指针，数据类型是`int *`。
    2. &a是一个内存地址，是指针a这个变量的内存地址，也是一个指针，是一个指向指针的指针。
    3. &a的数据类型是`int **`。

## 资料

printf用法大全，C语言printf格式控制符一览表

http://c.biancheng.net/view/159.html

## 总结

### 2021-05-08 10:56

写代码思路框架，耗费20多分钟。

理清IPC的代码思路时，有点费劲。思路不顺畅，我就会浪费时间，思维陷入非理性状态，无所事事，似乎在等待灵感。

为什么不顺畅？我其实在回忆于上神的代码思路，却不能完整地回忆出来。

为啥又顺畅了？我对自己说，不必受于上神的代码的束缚，我可以随心所欲地根据需求自由设计。

写完了代码框架（其实，就是几个函数），当初以为很多的东西，其实，没有多少。每个函数的细节，我都知道，却不愿写出来。

无论是边想边写，还是先想好再写，都会消耗思考时间。把写代码和思考分开，能减少难度（也许吧）。

写着写着，会思维混乱。若先想好了方案，思维不会混乱。

IPC逻辑，不比以前做过的业务需求多，真的更有技术含量吗？真的能使我更有竞争力吗？

IPC逻辑，不比以前做过的业务需求多。这是感觉。更准确的回答，是找出一个做过的需求，对比一下，

### 2021-05-08 15:04

理解嵌套指针的用法，耗费了58分，仍然没有理解。

代码是这样的：

```c
int printf(const char *fmt, ...)
{
	int i;
	char buf[256];

	va_list arg = (va_list)((char*)(&fmt) + 4); /*4是参数fmt所占堆栈中的大小*/
	i = vsprintf(buf, fmt, arg);
	buf[i] = 0;
	printx(buf);

	return i;
}

PUBLIC int vsprintf(char *buf, const char *fmt, va_list args)
{
	char*	p;

	va_list	p_next_arg = args;
	int	m;

	char	inner_buf[STR_DEFAULT_LEN];
	char	cs;
	int	align_nr;

	for (p=buf;*fmt;fmt++) {
		if (*fmt != '%') {
			*p++ = *fmt;
			continue;
		}
		else {		/* a format string begins */
			align_nr = 0;
		}

		fmt++;

		if (*fmt == '%') {
			*p++ = *fmt;
			continue;
		}
		else if (*fmt == '0') {
			cs = '0';
			fmt++;
		}
		else {
			cs = ' ';
		}
		while (((unsigned char)(*fmt) >= '0') && ((unsigned char)(*fmt) <= '9')) {
			align_nr *= 10;
			align_nr += *fmt - '0';
			fmt++;
		}

		char * q = inner_buf;
		memset(q, 0, sizeof(inner_buf));

		switch (*fmt) {
		case 'c':
			*q++ = *((char*)p_next_arg);
			p_next_arg += 4;
			break;
		case 'x':
			m = *((int*)p_next_arg);
			i2a(m, 16, &q);
			p_next_arg += 4;
			break;
		case 'd':
			m = *((int*)p_next_arg);
			if (m < 0) {
				m = m * (-1);
				*q++ = '-';
			}
			i2a(m, 10, &q);
			p_next_arg += 4;
			break;
		case 's':
			strcpy(q, (*((char**)p_next_arg)));
			q += strlen(*((char**)p_next_arg));
			p_next_arg += 4;
			break;
		default:
			break;
		}

		int k;
		for (k = 0; k < ((align_nr > strlen(inner_buf)) ? (align_nr - strlen(inner_buf)) : 0); k++) {
			*p++ = cs;
		}
		q = inner_buf;
		while (*q) {
			*p++ = *q++;
		}
	}

	*p = 0;

	return (p - buf);
}
```

`strcpy(q, (*((char**)p_next_arg)));`中的`(*((char**)p_next_arg))`能获取字符。理解不了。

我做了什么？

盯着代码看，依照原来的指针知识来自圆其说。

断点调试，观察值的变化，仍然没有看出规则。

大部分时间，盯着代码或打印出来的数据在看。

### 2021-05-08 15:35

再次理解嵌套指针，耗费了26分，都是有效时间。

也不能说完全理解了，我把通过断点调试打印出来的数据当作语法规则了。不问为什么，语法就是这样的。记住它，并且，我以后也可以这么用。

什么知识？

1. `int a`
   1. a的数据类型是int。
   2. &a的数据类型是`int *`。
2. int *b
   1. b的数据类型是`int *`。
   2. &b的数据类型是`int **`。
3. char *str
   1. str的数据类型是`char *`。
   2. &str的数据类型是`char **`。
4. char str
   1. str的数据类型是`char`。
   2. &str的数据类型是`char *`。

下面是用gdb断点打印出来的数据：

```shell
Breakpoint 1, main (argc=2, argv=0xffffd404) at point.c:18
18		char *str = "hello";
(gdb) s
19		char m = 'A';
(gdb) s
21		return 0;
(gdb) whatis str
type = char *
(gdb) whatis &str
type = char **
(gdb) ptype str
type = char *
(gdb) ptype &str
type = char **
(gdb) whatis m
type = char
(gdb) whatis &m
type = char *
(gdb) ptype m
type = char
(gdb) ptype &m
type = char *
```

gdb打印出的数据类型也不总是正确的，例如：

```shell
(gdb) ptype p_next_arg
type = char *
(gdb) ptype &p_next_arg
type = char **
(gdb) p &p_next_arg
$9 = (va_list *) 0x4fe3c <task_stack+97948>
(gdb) p p_next_arg
$10 = (va_list) 0x4ff88 <task_stack+98280> "\350?\003"
```

`ptype p_next_arg`打印出来的数据类型应该是`char **`。因为，p_next_arg的值是一个内存地址，而这个内存地址又指向另一块内存。换句话说，p_next_arg的值是一个指针，这个指针指向内存中的一个字节。

在具体场景中，p_next_arg的值是一个内存地址，在堆栈中。而堆栈中的那块内存中存储的，又是一个内存地址。

在这种语法问题上纠缠这么久，让我没心情继续写IPC的逻辑了。

### 2021-05-08 17:17

又验证嵌套指针，耗费42分，大部分时间是有效时间，运行例子，验证或摸索语法规则。

### 2021-05-08 17:47

把无符号整型数转为字符串，理解于上神的代码，耗费30分。

两个知识点：

1. 递归。
2. `char **`类型指针。理解起来比较烦。

### 2021-05-08 18:05

理解并验证`*(*ps)++`。太琐碎了。耗时32分。

代码是：

```c
char *str = "hello";
char **ps = &str;
```

用gdb断点调试的过程如下：

```shell
Breakpoint 1, main (argc=2, argv=0xffffd404) at point.c:18
18		char *str = "hello";
(gdb) s
19		char **ps = &str;
(gdb) s
21		return 0;
(gdb) p *(*ps)++
$47 = 104 'h'
(gdb) p *(*ps)++
$48 = 101 'e'
(gdb) p *((*ps)++)
$49 = 108 'l'
(gdb) p *((*ps)++)
$50 = 108 'l'
(gdb) p *((*ps)++)
$51 = 111 'o'
(gdb) p *((*ps)++)
$52 = 0 '\000'

(gdb) p *ps
$54 = 0x80485fd "hello"
(gdb) p *0x80485fd
$55 = 1819043176
(gdb) x /1wc  *0x80485fd
0x6c6c6568:	Cannot access memory at address 0x6c6c6568
(gdb) x /1wx 0x80485fd
0x80485fd:	0x6c6c6568
(gdb) x /1wc 0x80485fd
0x80485fd:	104 'h'
(gdb) x /1wc 0x80485fe
0x80485fe:	101 'e'
(gdb) p *ps
$56 = 0x80485fd "hello"
(gdb) p (*ps)+1
$57 = 0x80485fe "ello"
(gdb) p (*ps)+2
$58 = 0x80485ff "llo"
(gdb) p (*ps)+3
$59 = 0x8048600 "lo"
(gdb) p (*ps)+4
$60 = 0x8048601 "o"
```

内存分布示意图：

| 内存地址     | 0x00 | 0x02 | 0x03 | 0x04 | 0x05 | 0x06 | 0x07 | 0x08 |
| ------------ | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 变量名       | ps   | str  |      |      |      |      |      |      |
| 等价名称     |      | *ps  | **ps |      |      |      |      |      |
| 内存中的数据 | 0x01 | 0x03 | h    | e    | l    | l    | o    | \0   |
|              |      |      |      |      |      |      |      |      |

`*(*ps)++`，怎么理解？

1. `*ps`的值是`0x03`。
2. `(*ps)++`能依次得到：`0x03、0x04、0x05、0x06、0x07、0x08`。
3. 对一个内存地址（其实就是一个指针）使用`*`，获取或指向的是这个内存地址中的数据。
4. `*(*ps)++`等价于`*((*ps)++)`。
5. `*(*ps)++`的实质是：`*(0x03)`、`*(0x04)`、`*(0x05)`、`*(0x06)`、`*(0x07)`、`*(0x08)`。
6. 但是，直接使用`*(0x03)`、`*(0x04)`、`*(0x05)`、`*(0x06)`、`*(0x07)`、`*(0x08)`这种形式的语句，不符合C语言的语法。

### 2021-05-08 20:17

`sys_receive_msg`的业务逻辑，写文档，思维混乱，想不起来，只能看于上神的代码。有些操作，我思考后仍觉得没有必要。

比较烦躁。

耗费35分。

这个业务逻辑设计成这样，是个人喜好和IPC的要求。

分成了两部分，第一部分是检查能不能接收消息，第二部分是接收消息或不接收消息的具体措施。

