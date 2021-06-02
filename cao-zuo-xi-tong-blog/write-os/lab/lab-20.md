# 实现IPC

## 代码位置

调试函数

/home/cg/os/pegasus-os/v29

IPC函数

调试函数代码比较多，分成两份代码，避免出现错误难以发现。

/home/cg/os/pegasus-os/v30

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

## 写代码的思路

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

上面的写法是错误的。使用typedef，不能在结构体自身中使用自身声明成员。

正确的语法是：

```shell
struct Node{
        char name[20];
        int val;
        struct Node *next;
};
```



```c
#include <stdio.h>

struct Node{
        char name[20];
        int val;
        struct Node *next;
};

//typedef _Node Node;

void dump_node(struct Node *node);

void traverse(struct Node *node);

int main(int argc, char **argv)
{
        struct Node node2 = {"node2", 2, NULL};
        struct Node node1 = {"node1", 1, &node2};

        dump_node(&node1);
        dump_node(&node2);
        printf("\n\n");
        traverse(&node1);

        return 0;
}

void dump_node(struct Node *node)
{
        printf("node:%s,%d\n", node->name, node->val);
        return;
}
// 遍历队列
void traverse(struct Node *node)
{
        struct Node *current = node;
        while(current != NULL){
        //while(current != 0){
                dump_node(current);
                current = current->next;
        }
        return;
}
```

在上面的代码中，`while(current != NULL)`等价于`while(current != 0)`。

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

#### 通用函数

##### pid2proc

`Proc *pid2proc(int pid);`

根据进程ID获取进程表的指针。

> 返回值用Proc还是Proc *？

##### proc2pid

`int proc2pid(Proc *proc);`

根据进程表的指针计算进程ID。

##### get_line_addr

`int get_line_addr(int base_addr, int offset);`

获取进程中的变量的线性地址。

> 内存地址的数据类型，不是一个指针类型吗？直接用int能行吗？

##### seg2addr

`int seg2addr(int seg_selector)`

根据段选择子计算段基址。

#### sys_call给sys_write等函数传递参数

之前没想到，现在也不能心算出来。

这个问题是什么？

```c
void sys_printx(char *error_msg, int len, Proc *proc);
void sys_write(char *buf, int len, Proc *proc);
int sys_send_msg(Proc *sender, int receiver_pid, Message *msg);
// receive_msg 通过sys_call调用
int sys_receive_msg(Proc *receiver, int sender_id, Message *msg);
```

这些函数的参数不同，但在sys_call中却需要用相同的代码传递参数。这就是问题。

把代码修改成下面这样的：

```c
void sys_printx(char *error_msg, int len, Proc *proc);
void sys_write(char *buf, int len, Proc *proc);
int sys_send_msg(Message *msg, int receiver_pid, Proc *sender);
// receive_msg 通过sys_call调用
int sys_receive_msg(Message *msg, int sender_id, Proc *receiver);
```

从右到左，参数类型相同，有的函数参数特别多，可以再增加，不影响。

> 其实，不改成下面那样，也行。

要想不受影响，需要安排好多出来的参数的位置。

例如，test函数的参数特别多。

```c
// A
int test(Message *msg, int sender_id, Proc *receiver, int var1, int var2);
```



```c
// B
int test(int var1, int var2, Message *msg, int sender_id, Proc *receiver);
```

应该是A还是B？

> 不要心算。我没有能力心算出结果。
>
> 用笔算，这个问题，不难。

一、A

汇编伪代码：`push var2->push var1->push receiver->push sender_id->push msg`。

在test函数内获取参数，`[ebp+n]`，`msg-->[ebp+8]`，`sender_id-->[ebp+12]`，`receiver-->[ebp+16]`，`var1-->[ebp+20]`，`var2-->[ebp+24]`。

在sys_send_msg函数内获取参数，`[ebp+n]`，`msg-->[ebp+8]`，`sender_id-->[ebp+12]`，`receiver-->[ebp+16]`~~，`var1-->[ebp+20]`，`var2-->[ebp+24]`~~。不受多出来的参数影响，因为函数自己知道自己需要几个参数，其他的数据，它不获取。

> 收益于前几天想明白了这个问题。

二、B

汇编伪代码：`push receiver->push sender_id->push msg->push var2->push var1`。

在test函数内获取参数，`[ebp+n]`，`var1-->[ebp+8]`，`var2-->[ebp+12]`，`msg-->[ebp+16]`，`sender_id-->[ebp+20]`，`receiver-->[ebp+24]`。

在sys_send_msg函数内获取参数，`[ebp+n]`，~~`var1-->[ebp+8]`，`var2-->[ebp+12]`，~~`msg-->[ebp+16]`，`sender_id-->[ebp+20]`，`receiver-->[ebp+24]`。要想正确获取参数，第一个参数需要是[ebp+16]。这不符合C函数的"Calling conventions"。

根据调用规约，第一个参数只能是[ebp+8]或[ebp+4]。

能下结论了，使用A方案。



#### sys_send_msg

`int sys_send_msg(Proc *sender, int receiver_pid, Message *msg)`。

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
   1. sender->p_msg = msg;
   2. sender->p_send_to = receiver_pid;
   3. sender->p_flag = SENDING;
   4. 把sender中的消息放入receiver的p_send_queue。
   5. 阻塞sender。

被sys_call调用。

"receiver的消息来源是sender或ANY"，这个判断条件非常正确。发送消息时，sender的p_receive_from是0。

#### sys_receive_msg

`int sys_receive_msg(Proc *receiver, int sender_id, Message *msg)`。

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
   4. 解除sender的阻塞。
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

#### 整体流程

1. 要收发消息，被直接使用的函数是`send_rec`。
2. `send_rec`调用`send_msg`和`receive_msg`。
3. `send_rec`和`receive_msg`是系统调用函数，通过`sys_call`调用`sys_send_msg`和`sys_receive_msg`。

#### 难点

接收者的待发送队列。这是一个队列常规操作，我却觉得是难点。可悲！不过，我能写出思路，只是，我觉得它是难点。

我的方法和于上神不同，我使用的是结构体，于上神使用两个成员。

1. 移除队列的第一个元素，`p_send_queue = p_send_queue->next`。
2. 移除队列中的某个元素。

```c
target = 5;		// 目标进程ID
current = p_send_queue;
while(current != 0){
  	if(current->pid == target){
      	break;
    }
  	pre = current;
  	current = p_send_queue->next;
}

pre->next = current->next;
```



### 用IPC改写get_ticks

### TaskSys

系统任务。运行时，CPU的特权级是1。

1. 是一个进程。目前，所有进程体都是一个循环。
2. receive_msg，sender是ANY。系统任务，当然要能够接收来自任意来源的消息。
3. 从msg中获取
   1. source（谁请求系统调用，在后面会向这个进程返回数据）
   2. type，要求系统调用做什么，例如，获取时钟中断次数。
4. send_msg，receiver是前面获取的source，msg存储了要返回给source的数据。

### get_ticks

函数原型：`int get_ticks()`。

获取时钟中断次数。不必使用系统调用，而是通过IPC的方式，要求TaskSys返回时钟中断次数。

### sys_get_ticks

不需要了。

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

## 写代码

## 调试

### 通用函数

#### itoa

`itoa(int value, char *str, int base)`。

1. value是要被转换的数字。
2. str存储转换出来饿结果。
3. base是要把value转换成哪种进制的数字字符串，取值是2，8，10，16等。

我的疑问：在调用函数时，声明char *类型变量str，此时，str的值是 0x0。如果声明并赋值为数组，那么，多大的数组尺寸合适？

先把str声明并赋值为空字符串，等以后看别人怎么做。

```shell
main.c:1870:5: error: a label can only be part of a statement and a declaration is not a statement
     char *str;
     ^~~~
```



### 调试函数

```shell
main.c:425:2: error: expected identifier or '(' before 'else'
  else assertion_failure(#exp, __FILE__, __BASE_FILE__, __LINE__)
  ^~~~
main.c:425:25: error: stray '#' in program
  else assertion_failure(#exp, __FILE__, __BASE_FILE__, __LINE__)
                         ^
main.c: In function 'vsprintf':
main.c:1704:5: error: a label can only be part of a statement and a declaration is not a statement
     char *str = *(char **)next_arg;
     ^~~~
main.c:1711:5: error: a label can only be part of a statement and a declaration is not a statement
     char c = *(char *)next_arg;
     ^~~~
main.c: In function 'sys_printx':
main.c:1753:12: warning: assignment to 'int' from 'char *' makes integer from pointer without a cast [-Wint-conversion]
  line_addr = base + error_msg;
            ^
```

### ipc函数

```shell
gcc -g -c -fno-builtin -o kernel_main.o main.c -m32
main.c:254:2: error: unknown type name 'Proc'
        Proc *sender;
        ^
main.c:962:18: error: expected expression
                proc->header = {NULL, NULL};
                               ^
main.c:997:26: warning: comparison between pointer and integer ('Proc *' and 'int') [-Wpointer-integer-compare]
        assert(proc_ready_table == 894);
               ~~~~~~~~~~~~~~~~ ^  ~~~
main.c:484:25: note: expanded from macro 'assert'
#define assert(exp)  if(exp) ; \
                        ^~~
main.c:1691:14: warning: equality comparison result unused [-Wunused-comparison]
                        tty->tail == tty->buf;
                        ~~~~~~~~~~^~~~~~~~~~~
main.c:1691:14: note: use '=' to turn this equality comparison into an assignment
                        tty->tail == tty->buf;
                                  ^~
                                  =
main.c:1850:12: warning: incompatible pointer to integer conversion assigning to 'int' from 'char *' [-Wint-conversion]
        line_addr = base + error_msg;
                  ^ ~~~~~~~~~~~~~~~~
main.c:1938:7: warning: incompatible pointer to integer conversion initializing 'int' with an expression of type 'Message *'
      [-Wint-conversion]
                int msg_line_addr = base + msg;
                    ^               ~~~~~~~~~~
main.c:1941:28: warning: incompatible integer to pointer conversion passing 'int' to parameter of type 'void *' [-Wint-conversion]
                phycopy(receiver->p_msg, msg_line_addr, msg_size);
                                         ^~~~~~~~~~~~~
main.c:108:30: note: passing argument to parameter 'src' here
void Memcpy(void *dst, void *src, int size);
                             ^
main.c:1959:20: error: use of undeclared identifier 'NULL'
                while(current != NULL){
                                 ^
main.c:1963:15: error: expected expression
                pre->next = {sender, NULL};
                            ^
main.c:1981:32: error: use of undeclared identifier 'NULL'
                if(receiver->header->next != NULL){
                                             ^
main.c:1982:22: warning: incompatible pointer types passing 'struct MsgSender *' to parameter of type 'Proc *'
      [-Wincompatible-pointer-types]
                        p_from = proc2pid(receiver->header->next);
                                          ^~~~~~~~~~~~~~~~~~~~~~
main.c:1921:20: note: passing argument to parameter 'proc' here
int proc2pid(Proc *proc)
                   ^
main.c:1994:21: error: use of undeclared identifier 'NULL'
                        while(current != NULL){
                                         ^
main.c:1995:17: error: no member named 'pid' in 'struct MsgSender'
                                if(current->pid == sender_id){
                                   ~~~~~~~  ^
main.c:2020:21: warning: incompatible pointer to integer conversion initializing 'int' with an expression of type 'Message *'
      [-Wint-conversion]
                int msg_line_addr = base + msg;
                    ^               ~~~~~~~~~~
main.c:2023:25: warning: incompatible integer to pointer conversion passing 'int' to parameter of type 'void *' [-Wint-conversion]
                phycopy(msg_line_addr, p_from_proc->p_msg, msg_size);
                        ^~~~~~~~~~~~~
main.c:108:19: note: passing argument to parameter 'dst' here
void Memcpy(void *dst, void *src, int size);
                  ^
main.c:2042:31: error: use of undeclared identifier 'sender_pid'; did you mean 'sender_id'?
                        receiver->p_receive_from = sender_pid;
                                                   ^~~~~~~~~~
                                                   sender_id
main.c:1972:39: note: 'sender_id' declared here
int sys_receive_msg(Message *msg, int sender_id, Proc *receiver)
                                      ^
8 warnings and 8 errors generated.
make: *** [kernel.bin] Error 1
```

修改之后，

```shell
main.c:969:18: error: expected expression
                proc->header = {-1, NULL};
                               ^
main.c:1004:26: warning: comparison between pointer and integer ('Proc *' and 'int') [-Wpointer-integer-compare]
        assert(proc_ready_table == 894);
               ~~~~~~~~~~~~~~~~ ^  ~~~
main.c:491:25: note: expanded from macro 'assert'
#define assert(exp)  if(exp) ; \
                        ^~~
main.c:1698:14: warning: equality comparison result unused [-Wunused-comparison]
                        tty->tail == tty->buf;
                        ~~~~~~~~~~^~~~~~~~~~~
main.c:1698:14: note: use '=' to turn this equality comparison into an assignment
                        tty->tail == tty->buf;
                                  ^~
                                  =
main.c:1857:12: warning: incompatible pointer to integer conversion assigning to 'int' from 'char *' [-Wint-conversion]
        line_addr = base + error_msg;
                  ^ ~~~~~~~~~~~~~~~~
main.c:1945:7: warning: incompatible pointer to integer conversion initializing 'int' with an expression of type 'Message *'
      [-Wint-conversion]
                int msg_line_addr = base + msg;
                    ^               ~~~~~~~~~~
main.c:1948:28: warning: incompatible integer to pointer conversion passing 'int' to parameter of type 'void *' [-Wint-conversion]
                phycopy(receiver->p_msg, msg_line_addr, msg_size);
                                         ^~~~~~~~~~~~~
main.c:108:30: note: passing argument to parameter 'src' here
void Memcpy(void *dst, void *src, int size);
                             ^
main.c:1971:15: error: expected expression
                pre->next = {sender_id, 0x0};
                            ^
main.c:2028:21: warning: incompatible pointer to integer conversion initializing 'int' with an expression of type 'Message *'
      [-Wint-conversion]
                int msg_line_addr = base + msg;
                    ^               ~~~~~~~~~~
main.c:2031:25: warning: incompatible integer to pointer conversion passing 'int' to parameter of type 'void *' [-Wint-conversion]
                phycopy(msg_line_addr, p_from_proc->p_msg, msg_size);
                        ^~~~~~~~~~~~~
main.c:108:19: note: passing argument to parameter 'dst' here
void Memcpy(void *dst, void *src, int size);
                  ^
main.c:2050:31: error: use of undeclared identifier 'sender_pid'; did you mean 'sender_id'?
                        receiver->p_receive_from = sender_pid;
                                                   ^~~~~~~~~~
                                                   sender_id
main.c:1980:39: note: 'sender_id' declared here
int sys_receive_msg(Message *msg, int sender_id, Proc *receiver)
                                      ^
7 warnings and 3 errors generated.
make: *** [kernel.bin] Error 1
```

剩下的全部是警告，暂时无视：

```shell
main.c: In function 'TestA':
main.c:1007:26: warning: comparison between pointer and integer
  assert(proc_ready_table == 894);
                          ^~
main.c:491:25: note: in definition of macro 'assert'
 #define assert(exp)  if(exp) ; \
                         ^~~
main.c: In function 'sys_printx':
main.c:1860:12: warning: assignment to 'int' from 'char *' makes integer from pointer without a cast [-Wint-conversion]
  line_addr = base + error_msg;
            ^
main.c: In function 'sys_send_msg':
main.c:1948:23: warning: initialization of 'int' from 'Message *' {aka 'struct <anonymous> *'} makes integer from pointer without a cast [-Wint-conversion]
   int msg_line_addr = base + msg;
                       ^~~~
main.c:1951:28: warning: passing argument 2 of 'Memcpy' makes pointer from integer without a cast [-Wint-conversion]
   phycopy(receiver->p_msg, msg_line_addr, msg_size);
                            ^~~~~~~~~~~~~
main.c:108:30: note: expected 'void *' but argument is of type 'int'
 void Memcpy(void *dst, void *src, int size);
                        ~~~~~~^~~
main.c: In function 'sys_receive_msg':
main.c:2033:37: warning: initialization of 'int' from 'Message *' {aka 'struct <anonymous> *'} makes integer from pointer without a cast [-Wint-conversion]
                 int msg_line_addr = base + msg;
                                     ^~~~
main.c:2036:25: warning: passing argument 1 of 'Memcpy' makes pointer from integer without a cast [-Wint-conversion]
                 phycopy(msg_line_addr, p_from_proc->p_msg, msg_size);
                         ^~~~~~~~~~~~~
main.c:108:19: note: expected 'void *' but argument is of type 'int'
 void Memcpy(void *dst, void *src, int size);
```

### 用IPC改写get_ticks

运行的时候，出现不明原因错误：

```shell
00014033826i[BIOS  ] Booting from 0000:7c00
00287112033e[CPU0  ] fetch_raw_descriptor: LDT: index (10c7) 218 > limit (f)
00287222295i[CPU0  ] WARNING: HLT instruction with IF=0!
```

一、是求进程中的msg的物理地址出错了吗？

在get_ticks、sys_send_msg、sys_receive_msg设置断点。

发现了：

1. Message *msg; 创建的msg的内存地址是0x0。神奇。
2. 修改为Message msg后正常了。



二、奇怪，出现死锁了

```c
(gdb) p proc_table[1]
$16 = {s_reg = {gs = 57, fs = 13, es = 13, ds = 13, edi = 0, esi = 0, ebp = 1269372, kernel_esp = 1363232, ebx = 2, edx = 0,
    ecx = 1269340, eax = 0, eip = 206554, cs = 5, eflags = 4626, esp = 1269312, ss = 13}, ldt_selector = 80, ldts = {{
      seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 187 '\273',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
      seg_base_middle = 0 '\000', seg_attr1 = 179 '\263', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 1,
  ticks = 15, priority = 15, tty_index = 0, name = '\000' <repeats 19 times>, p_flag = 1, p_msg = 0x135e5c <proc_stack+988>,
  p_send_to = 2, p_receive_from = 0, header = 0x0}
(gdb) p proc_table[2]
$17 = {s_reg = {gs = 57, fs = 15, es = 15, ds = 15, edi = 1269572, esi = 1269453, ebp = 1269788, kernel_esp = 1363376, ebx = 1, edx = 10,
    ecx = 1269824, eax = 0, eip = 206554, cs = 7, eflags = 534, esp = 1269744, ss = 15}, ldt_selector = 88, ldts = {{
      seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 251 '\373',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
      seg_base_middle = 0 '\000', seg_attr1 = 243 '\363', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 2,
  ticks = 7, priority = 8, tty_index = 0, name = '\000' <repeats 19 times>, p_flag = 1, p_msg = 0x136040 <proc_stack+1472>,
  p_send_to = 1, p_receive_from = 0, header = 0x0}
(gdb) p proc_table[3]
$18 = {s_reg = {gs = 57, fs = 15, es = 15, ds = 15, edi = 0, esi = 0, ebp = 1270300, kernel_esp = 1363520, ebx = 1, edx = 0,
    ecx = 1270336, eax = 0, eip = 206570, cs = 7, eflags = 534, esp = 1270256, ss = 15}, ldt_selector = 96, ldts = {{
      seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 251 '\373',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
      seg_base_middle = 0 '\000', seg_attr1 = 243 '\363', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 3,
  ticks = 8, priority = 8, tty_index = 1, name = '\000' <repeats 19 times>, p_flag = 2, p_msg = 0x136240 <proc_stack+1984>,
  p_send_to = 0, p_receive_from = 1, header = 0x0}
(gdb) p proc_table[4]
$19 = {s_reg = {gs = 57, fs = 15, es = 15, ds = 15, edi = 0, esi = 0, ebp = 1270812, kernel_esp = 1363664, ebx = 1, edx = 0,
    ecx = 1270848, eax = 0, eip = 206554, cs = 7, eflags = 534, esp = 1270768, ss = 15}, ldt_selector = 104, ldts = {{
      seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 251 '\373',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
      seg_base_middle = 0 '\000', seg_attr1 = 243 '\363', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 4,
  ticks = 8, priority = 8, tty_index = 2, name = '\000' <repeats 19 times>, p_flag = 1, p_msg = 0x136440 <proc_stack+2496>,
  p_send_to = 1, p_receive_from = 0, header = 0x0}
```



```shell
(gdb) p proc_table[1]
$16 = {s_reg = {gs = 57, fs = 13, es = 13, ds = 13, edi = 0, esi = 0, ebp = 1269372, kernel_esp = 1363232, ebx = 2, edx = 0,
    ecx = 1269340, eax = 0, eip = 206554, cs = 5, eflags = 4626, esp = 1269312, ss = 13}, ldt_selector = 80, ldts = {{
      seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 187 '\273',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
      seg_base_middle = 0 '\000', seg_attr1 = 179 '\263', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 1,
  ticks = 15, priority = 15, tty_index = 0, name = '\000' <repeats 19 times>, p_flag = 1, p_msg = 0x135e5c <proc_stack+988>,
  p_send_to = 2, p_receive_from = 0, header = 0x0}
(gdb) p proc_table[2]
$17 = {s_reg = {gs = 57, fs = 15, es = 15, ds = 15, edi = 1269572, esi = 1269453, ebp = 1269788, kernel_esp = 1363376, ebx = 1, edx = 10,
    ecx = 1269824, eax = 0, eip = 206554, cs = 7, eflags = 534, esp = 1269744, ss = 15}, ldt_selector = 88, ldts = {{
      seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 251 '\373',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
      seg_base_middle = 0 '\000', seg_attr1 = 243 '\363', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 2,
  ticks = 7, priority = 8, tty_index = 0, name = '\000' <repeats 19 times>, p_flag = 1, p_msg = 0x136040 <proc_stack+1472>,
  p_send_to = 1, p_receive_from = 0, header = 0x0}
```

观察p_send_to。

1. TaskSys运行
   1. receive
   2. 被阻塞
2. A运行
   1. send
   2. 解除TaskSys阻塞
   3. A运行receive，被阻塞
   4. TaskSys运行send，解除A的阻塞
   5. TaskSys运行receive，被阻塞
   6. A运行send，解除T的阻塞

三、

```shell
main.c:976:15: error: invalid type argument of '->' (have 'struct MsgSender')
   proc->header->sender_pid = -1;
               ^~
main.c:977:15: error: invalid type argument of '->' (have 'struct MsgSender')
   proc->header->next = 0;
               ^~
main.c: In function 'sys_printx':
main.c:1876:12: warning: assignment to 'int' from 'char *' makes integer from pointer without a cast [-Wint-conversion]
  line_addr = base + error_msg;
            ^
main.c: In function 'sys_send_msg':
main.c:1964:23: warning: initialization of 'int' from 'Message *' {aka 'struct <anonymous> *'} makes integer from pointer without a cast [-Wint-conversion]
   int msg_line_addr = base + msg;
                       ^~~~
main.c:1967:28: warning: passing argument 2 of 'Memcpy' makes pointer from integer without a cast [-Wint-conversion]
   phycopy(receiver->p_msg, msg_line_addr, msg_size);
                            ^~~~~~~~~~~~~
main.c:108:30: note: expected 'void *' but argument is of type 'int'
 void Memcpy(void *dst, void *src, int size);
                        ~~~~~~^~~
main.c:1985:38: error: 'receiver' is a pointer; did you mean to use '->'?
   struct MsgSender current = receiver.header;
                                      ^
                                      ->
main.c:1986:34: error: 'receiver' is a pointer; did you mean to use '->'?
   struct MsgSender pre = receiver.header;
                                  ^
                                  ->
main.c:1987:17: error: invalid operands to binary != (have 'struct MsgSender' and 'int')
   while(current != 0x0){
                 ^~
main.c:1989:12: error: incompatible types when assigning to type 'struct MsgSender' from type 'struct MsgSender *'
    current = current.next;
            ^
main.c: In function 'sys_receive_msg':
main.c:2015:12: error: 'receiver' is a pointer; did you mean to use '->'?
    receiver.header->next = receiver.header->next->next;
            ^
            ->
main.c:2015:36: error: 'receiver' is a pointer; did you mean to use '->'?
    receiver.header->next = receiver.header->next->next;
                                    ^
                                    ->
main.c:2025:39: error: 'receiver' is a pointer; did you mean to use '->'?
    struct MsgSender current = receiver.header;
                                       ^
                                       ->
main.c:2026:35: error: 'receiver' is a pointer; did you mean to use '->'?
    struct MsgSender pre = receiver.header;
                                   ^
                                   ->
main.c:2027:18: error: invalid operands to binary != (have 'struct MsgSender' and 'int')
    while(current != 0x0){
                  ^~
main.c:2028:15: error: invalid type argument of '->' (have 'struct MsgSender')
     if(current->sender_pid == sender_pid){
               ^~
main.c:2036:13: error: incompatible types when assigning to type 'struct MsgSender' from type 'struct MsgSender *'
     current = current.next;
             ^
main.c:2053:37: warning: initialization of 'int' from 'Message *' {aka 'struct <anonymous> *'} makes integer from pointer without a cast [-Wint-conversion]
                 int msg_line_addr = base + msg;
                                     ^~~~
main.c:2056:25: warning: passing argument 1 of 'Memcpy' makes pointer from integer without a cast [-Wint-conversion]
                 phycopy(msg_line_addr, p_from_proc->p_msg, msg_size);
                         ^~~~~~~~~~~~~
main.c:108:19: note: expected 'void *' but argument is of type 'int'
 void Memcpy(void *dst, void *src, int size);
             ~~~~~~^~~
make: *** [Makefile:32: kernel.bin] Error 1
```



```shell
main.c:1987:9: error: invalid type argument of unary '*' (have 'struct MsgSender')
   while(*current != 0x0){
         ^~~~~~~~
main.c: In function 'sys_receive_msg':
main.c:2027:10: error: invalid type argument of unary '*' (have 'struct MsgSender')
    while(*current != 0x0){
          ^~~~~~~~
```



只剩下警告：

```shell
main.c: In function 'sys_printx':
main.c:1876:12: warning: assignment to 'int' from 'char *' makes integer from pointer without a cast [-Wint-conversion]
  line_addr = base + error_msg;
            ^
main.c: In function 'sys_send_msg':
main.c:1964:23: warning: initialization of 'int' from 'Message *' {aka 'struct <anonymous> *'} makes integer from pointer without a cast [-Wint-conversion]
   int msg_line_addr = base + msg;
                       ^~~~
main.c:1967:28: warning: passing argument 2 of 'Memcpy' makes pointer from integer without a cast [-Wint-conversion]
   phycopy(receiver->p_msg, msg_line_addr, msg_size);
                            ^~~~~~~~~~~~~
main.c:108:30: note: expected 'void *' but argument is of type 'int'
 void Memcpy(void *dst, void *src, int size);
                        ~~~~~~^~~
main.c: In function 'sys_receive_msg':
main.c:2055:37: warning: initialization of 'int' from 'Message *' {aka 'struct <anonymous> *'} makes integer from pointer without a cast [-Wint-conversion]
                 int msg_line_addr = base + msg;
                                     ^~~~
main.c:2058:25: warning: passing argument 1 of 'Memcpy' makes pointer from integer without a cast [-Wint-conversion]
                 phycopy(msg_line_addr, p_from_proc->p_msg, msg_size);
                         ^~~~~~~~~~~~~
main.c:108:19: note: expected 'void *' but argument is of type 'int'
 void Memcpy(void *dst, void *src, int size);
```



```shell
main.c:259:19: error: field 'next' has incomplete type
  struct MsgSender next;
                   ^~~~
main.c: In function 'sys_printx':
main.c:1883:12: warning: assignment to 'int' from 'char *' makes integer from pointer without a cast [-Wint-conversion]
  line_addr = base + error_msg;
            ^
main.c: In function 'sys_send_msg':
main.c:1971:23: warning: initialization of 'int' from 'Message *' {aka 'struct <anonymous> *'} makes integer from pointer without a cast [-Wint-conversion]
   int msg_line_addr = base + msg;
                       ^~~~
main.c:1974:28: warning: passing argument 2 of 'Memcpy' makes pointer from integer without a cast [-Wint-conversion]
   phycopy(receiver->p_msg, msg_line_addr, msg_size);
                            ^~~~~~~~~~~~~
main.c:108:30: note: expected 'void *' but argument is of type 'int'
 void Memcpy(void *dst, void *src, int size);
                        ~~~~~~^~~
main.c:1995:17: error: invalid operands to binary != (have 'struct MsgSender' and 'int')
   while(current != 0x0){
                 ^~
main.c: In function 'sys_receive_msg':
main.c:2036:18: error: invalid operands to binary != (have 'struct MsgSender' and 'int')
    while(current != 0x0){
                  ^~
main.c:2062:37: warning: initialization of 'int' from 'Message *' {aka 'struct <anonymous> *'} makes integer from pointer without a cast [-Wint-conversion]
                 int msg_line_addr = base + msg;
                                     ^~~~
main.c:2065:25: warning: passing argument 1 of 'Memcpy' makes pointer from integer without a cast [-Wint-conversion]
                 phycopy(msg_line_addr, p_from_proc->p_msg, msg_size);
                         ^~~~~~~~~~~~~
main.c:108:19: note: expected 'void *' but argument is of type 'int'
 void Memcpy(void *dst, void *src, int size);
             ~~~~~~^~~
make: *** [Makefile:32: kernel.bin] Error 1
```

四、把发送消息队列改成和于上神一样的吧。

1. 两三分钟内，我还想不出应该怎么实现。
2. 不用他的方案。同样出现指针变量的内存地址是0的问题。

五、耗费了非常非常多的时间，仍然没有解决问题。

A.只能打印部分数据

1. 在其他版本测试打印数据，例如v29、v28

B.死锁

C.发送消息的队列

## 工具

### vim+ctags+Taglist

#### centos7安装

```shell
cd /usr/share/vim/vimfiles
wget https://nchc.dl.sourceforge.net/project/vim-taglist/vim-taglist/4.6/taglist_46.zip
unzip taglist_46.zip
```

vim ~/.vimrc

```shell
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" CTags的设定 
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
"let Tlist_Sort_Type = "name" " 按照名称排序 
let Tlist_Enable_Fold_Column = 0
let Tlist_Use_Right_Window = 0 " 在左侧显示窗口 
let Tlist_Compart_Format = 1 " 压缩方式 
let Tlist_Exist_OnlyWindow = 1 " 如果只有一个buffer，kill窗口也kill掉buffer 
let Tlist_File_Fold_Auto_Close = 0 " 不要关闭其他文件的tags 
let Tlist_Enable_Fold_Column = 1 " 要显示折叠树 
autocmd FileType java set tags+=D:\tools\java\tags 
"autocmd FileType h,cpp,cc,c set tags+=D:\tools\cpp\tags 
"let Tlist_Show_One_File=1 "不同时显示多个文件的tag，只显示当前文件的
"设置tags 
set tags=tags 
"set autochdir 

""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
"其他东东
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
"默认打开Taglist 
let Tlist_Auto_Open=1 
"""""""""""""""""""""""""""""" 
" Tag list (ctags) 
"""""""""""""""""""""""""""""""" 
let Tlist_Ctags_Cmd = '/usr/bin/ctags'
```

#### macbook安装

```shell
cd ~/.vim
wget https://nchc.dl.sourceforge.net/project/vim-taglist/vim-taglist/4.6/taglist_46.zip
unzip taglist_46.zip

```



`vim ~/.vimrc`，增加下面的配置：

```shell
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" CTags的设定 
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
"let Tlist_Sort_Type = "name" " 按照名称排序 
let Tlist_Enable_Fold_Column = 0
let Tlist_Use_Right_Window = 0 " 在左侧显示窗口 
let Tlist_Compart_Format = 1 " 压缩方式 
let Tlist_Exist_OnlyWindow = 1 " 如果只有一个buffer，kill窗口也kill掉buffer 
let Tlist_File_Fold_Auto_Close = 0 " 不要关闭其他文件的tags 
let Tlist_Enable_Fold_Column = 1 " 要显示折叠树 
"autocmd FileType java set tags+=D:\tools\java\tags 
"autocmd FileType h,cpp,cc,c set tags+=D:\tools\cpp\tags 
"let Tlist_Show_One_File=1 "不同时显示多个文件的tag，只显示当前文件的
"设置tags 
set tags=tags 
"set autochdir 

""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
"其他东东
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
"默认打开Taglist 
let Tlist_Auto_Open=1 
"""""""""""""""""""""""""""""" 
" Tag list (ctags) 
"""""""""""""""""""""""""""""""" 
let Tlist_Ctags_Cmd = '/usr/bin/local/ctags'
```

遇到问题：

```shell
Taglist: Failed to generate tags for /Users/cg/data/code/os/pegasus-os/v29/main.c
/Library/Developer/CommandLineTools/usr/bin/ctags: illegal option -- -^@usage: ctags [-BFadtuwvx] [-f tagsfile] file ...^
@
```

解决：

```

mkdir ~/soft
cd ~/soft
wget https://udomain.dl.sourceforge.net/project/ctags/ctags/5.8/ctags58.zip

```

卸载旧ctags:

```shell
chugangdeMacBook-Pro:my-note-book cg$ ls -al /usr/local/bin/ctags
lrwxr-xr-x 1 cg admin 31  5  7 09:32 /usr/local/bin/ctags -> ../Cellar/ctags/5.8_1/bin/ctags
chugangdeMacBook-Pro:my-note-book cg$ brew uninstall ctags
Uninstalling /usr/local/Cellar/ctags/5.8_1... (9 files, 344.5KB)
chugangdeMacBook-Pro:my-note-book cg$ ls -al /usr/local/bin/ctags
ls: cannot access '/usr/local/bin/ctags': No such file or directory
```

重新安装：

```shell
brew install ctags-exuberant
```

从源码编译安装ctags-exuberant，太困难：

1. 源码包中的方法，在我的mac上出错。
2. 网络资料“./configure"等，行不通。源码包中没有configure文件。

目前，我的mac上使用的配置文件是：

```shell
set number

syntax on


" plug-vim
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" Specify a directory for plugins
call plug#begin('~/.vim/plugged')

Plug 'iamcco/mathjax-support-for-mkdp'
Plug 'iamcco/markdown-preview.vim'

"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" CTags的设定
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
"let Tlist_Sort_Type = "name" " 按照名称排序
let Tlist_Enable_Fold_Column = 0
let Tlist_Use_Right_Window = 1 " 在右侧显示窗口
let Tlist_Compart_Format = 1 " 压缩方式
let Tlist_Exist_OnlyWindow = 1 " 如果只有一个buffer，kill窗口也kill掉buffer
let Tlist_File_Fold_Auto_Close = 0 " 不要关闭其他文件的tags
let Tlist_Enable_Fold_Column = 1 " 要显示折叠树
"autocmd FileType java set tags+=D:\tools\java\tags
autocmd FileType java set tags+=/Users/cg/data/code/os/pegasus-os/v29/tags
"autocmd FileType h,cpp,cc,c set tags+=D:\tools\cpp\tags
"let Tlist_Show_One_File=1 "不同时显示多个文件的tag，只显示当前文件的
"设置tags
set tags=tags
"set autochdir

""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
"其他东东
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
"默认打开Taglist
let Tlist_Auto_Open=1
""""""""""""""""""""""""""""""
" Tag list (ctags)
""""""""""""""""""""""""""""""""
"let Tlist_Ctags_Cmd = '/usr/bin/ctags'
let Tlist_Ctags_Cmd='/usr/local/bin/ctags'
call plug#end()
```



#### 基本使用

![image-20210509110549482](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210509110549482.png)

```shell
#在A-B窗口之间切换
control + ww
#打开文件并定位到函数
vim -t 函数名
#在Taglist窗口，定位到某个tag后
o#在新窗口打开tag所在的文件
ENTER键#在当前窗口切换到tag所在的文件


o 在一个新打开的窗口中显示光标下tag
显示光标下tag的原型定义
u 更新taglist窗口中的tag
s 更改排序方式，在按名字排序和按出现顺序排序间切换
x taglist窗口放大和缩小，方便查看较长的tag。没看出效果。
+ 打开一个折叠，同zo
- 将tag折叠起来，同zc
* 打开所有的折叠，同zR
= 将所有tag折叠起来，同zM
[[ 跳到前一个类别，例如，从function跳到typedef
]] 跳到后一个类别，同上
q 关闭taglist窗口

可以用“:TlistOpen”打开taglist窗口，用“:TlistClose”关闭taglist窗口。或者使用“:TlistToggle”在打开和关闭间切换。


vims使用
:e	强制刷新文件内容
:e! 强制丢掉本地修改，从磁盘加载文件
```

### Mac sublime Text3快捷键

```html
Sublime Text 常用快捷键（MAC 下）

符号说明

⌘：command 
⌃：control 
⌥：option 
⇧：shift 
↩：enter 
⌫：delete 
打开/关闭/前往

快捷键 功能 
⌘⇧N 打开一个新的sublime窗口 
⌘N 新建文件 
⌘⇧W 关闭sublime，关闭所有文件 
⌘W 关闭当前文件 
⌘P 跳转、前往文件、前往项目、命令提示、前往method等等（Goto anything） 
⌘⇧T 重新打开最近关闭的文件 
⌘T 前往文件 
⌘⌃P 前往项目 
⌘R 前往method 
⌘⇧P 命令提示 
⌃G 前往行 
⌘KB 开关侧栏 
⌃` 打开控制台 
⌃- 光标跳回上一个位置 
⌃⇧- 光标恢复位置 
编辑

快捷键 功能 
⌘A 全选 
⌘L 选择行（重复按下将下一行加入选择） 
⌘D 选择词（重复按下时多重选择相同的词进行多重编辑） 
⌃⇧M 选择括号的内容 
⌘⇧↩ 在当前行前插入新行 
⌘↩ 在当前行后插入新行 
⌃⇧K 删除行 
⌘KK 从光标处删除至行尾 
⌘K⌫ 从光标处删除至行首 
⌘⇧D 复制（多）行 
⌘J 合并（多）行 
⌘KU 改为大写 
⌘KL 改为小写 
⌘C 复制 
⌘X 剪切 
⌘V 粘贴 
⌘/ 注释 
⌘⌥/ 块注释 
⌘Z 撤销 
⌘Y 恢复撤销 
⌘⇧V 粘贴并自动缩进 
⌘⌥V 从历史中选择粘贴 
⌃M 跳转至对应的括号 
⌘U 软撤销（可撤销光标移动） 
⌘⇧U 软重做（可重做光标移动） 
⌘⇧S 保存所有文件 
⌘] 向右缩进 
⌘[ 向左缩进 
⌘⌥T 特殊符号集 
⌘⇧L 将选区转换成多个单行选区 
查找/替换

快捷键 功能 
⌘f 查找 
⌘⌥f 查找并替换 
⌘⌥g 查找下一个符合当前所选的内容 
⌘⌃g 查找所有符合当前选择的内容进行多重编辑 
⌘⇧F 在所有打开的文件中进行查找 
拆分窗口/标签页

快捷键 功能 
⌘⌥[1,2,3,4] 单列、双列、三列、四列 
⌘⌥5 网格（4组） 
⌃[1,2,3,4] 焦点移动到相应的组（分屏编号） 
⌃⇧[1,2,3,4] 将当前文件移动到相应的组（分屏编号） 
⌘[1,2,3,4] 选择相应的标签页 
快捷操作

快捷键 功能 
⌘⌃上下键 两行交换位置 
⌘KB 显示/隐藏侧边
```

### gdb

跳出循环：until

不进入函数执行：n

## 资料

printf用法大全，C语言printf格式控制符一览表

http://c.biancheng.net/view/159.html



电脑文件索引：

/Users/cg/data/code/note/c/jian-zhi-offer-cpp/tree

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

### 2021-05-09 08:29

回顾sys_send_msg、sys_receive_msg，写出移除队列中某个元素的代码。

耗费29分。还算正常。

阻塞sender、receiver，为什么要把进程的p_msg指向msg，我明白了。在复制消息时会用到。

原来的设计必须修改，只有深入到细节才能确定所有的细节。在工作中，遇到不允许修改细节的上级，那就有点不好办。必须在做之前想好所有细节。

### 2021-05-09 09:29

消息结构体，于上神的设计着实奇怪。根据目前的需求，我怎么也设计不出来这样的结构体，也不理解如此设计有什么作用。

耗费10分。

IPC的函数设计，我没有按照于上神的思路，看他的思路，和我的思路混合，思维又有点混乱。

先实现功能，孰优孰劣，那是我以后应该关注的问题。

消息体，盯着代码看。

目前，消息体只有三个必需成员：

1. source：系统调用进程ID。
2. type：source要干什么，比如，获取ticks。
3. val：传递结果。比如，ticks是5。

### 2021-05-09 12:59

在macbooks上安装Taglist，耗费了2个小时。

安装软件、配置环境等运维工作，非常耗费时间，很累，体力劳动。

有什么诀窍呢？无非就是搜索文档，然后照着做。这个方法不行，就再换一个。

以前，耗费了大量时间从源码编译安装PHP、MySQL等，对我升职加薪毫无帮助。

非必要不做环境配置、工具安装这类。SRE工作不等同这类吧？

折腾了太久运维，没精力和心情写代码了。

### 2021-05-09 17:48

写完调试函数后，调试vsprintf。

懒得查看代码找错误，直接编译，一大堆报错：

1. A函数用到了B函数，B函数却在A函数之后声明。
2. assert宏定义错误。
3. `char *str = *(char **)next_arg`，语法不正确。我理解不了。
4. 一些警告。
5. Strlen，问题。---最耗费时间，数据问题，代码无问题。

耗费1小时30分，不包括写代码的时间。还算正常。

比写文档有趣，不头晕。

### 2021-05-09 18:26

想测试assert等，却被自己弄出来的一个小改动坑死。突然想到，是那个改动导致问题。

修改了 k_reenter 的数据类型，没有修改其他地方。

耗费了34分。

### 2021-05-09 19:01

耗费了34分。

又是k_reenter。

### 2021-05-09 23:19

k_reenter问题。

耗费了1小时38分。

经验是：心算只会白白浪费时间，用笔算；不要盯着代码看，要计算、推理。像下面那样假设、推理。

Dec [k_reenter]的位置

1. 第一个进程启动，k = max
2. 第一个中断，k++，k = 0
3. 从中断中恢复，k--，k = max



自增k，在sti后还是前？

1. 后。
   1. 开启中断后，此中断未处理结束前，迅速发送了一个中断，可能在k自增前，此时，新中断将不会被视作中断重入。
2. 应该是在sti前。

```shell
Breakpoint 2, clock_handler () at main.c:1097
1097	{
(gdb) p k_reenter
$4 = 0
(gdb) info br
Num     Type           Disp Enb Address    What
1       breakpoint     keep y   0x00031685 in TaskTTY at main.c:1254
	breakpoint already hit 2 times
2       breakpoint     keep y   0x000313ee in clock_handler at main.c:1097
	breakpoint already hit 2 times
3       breakpoint     keep y   0x000311b8 in TestA at main.c:909
(gdb) del 1
(gdb) del 2
(gdb) c
Continuing.

Breakpoint 3, TestA () at main.c:909
909		unsigned int i = 0;
(gdb) p k_reenter
$5 = 4294967295
(gdb) b sys_printx
Breakpoint 4 at 0x32038: file main.c, line 1753.
(gdb) c
Continuing.

Breakpoint 4, sys_printx (error_msg=0x134f1c <proc_stack+1180> "", proc=0x10) at main.c:1753
1753		if(k_reenter == 0){
(gdb) p k_reenter
$6 = 0
(gdb)
```

包含了：

1. 查看断点。
2. 删除断点。
3. k_reenter是正确的。
4. sys_printx的proc不正确

sys_printx的proc不正确

使用bochs断点调试

1. 在write_debug设置断点，进入sys_call

```
sed -i "s/xchg/;xchg/g" *asm

```

```
<bochs:17> xp /1wx 0x30:(0x0013462c+12)
[bochs]:
0x0000000000134638 <bogus+       0>:	0x00000007

<bochs:24> xp /1wx 0xd:0x00134a60
[bochs]:
0x0000000000134a60 <bogus+       0>:	0x0014bcc8

rax: 00000000_00000001
rbx: 00000000_00000007
rcx: 00000000_00134d3c

<bochs:34> xp /1wx 0x30:(0x0013462c+16)
[bochs]:
0x000000000013463c <bogus+       0>:	0x0014bcc8
<bochs:35> xp /1wx 0x30:(0x0013462c+12)
[bochs]:
0x0000000000134638 <bogus+       0>:	0x00000007
<bochs:36> xp /1wx 0x30:(0x0013462c+8)
[bochs]:
0x0000000000134634 <bogus+       0>:	0x00134d3c



```



xp /1wx 0x30:(0x0013462c+16)

xp /1wx 0x30:(0x0013462c+12)

xp /1wx 0x30:(0x0013462c+8)



xp /1wc 0x30:(0x0013462c+8)



看不出问题。不知道Proc *proc在汇编中打印出来是什么。

2. 用gdb查看sys_write和sys_printx的参数中的proc的差异。

$3 = ";i==2\000 error in\nfile:main.c\000\nbase_file:main.c\000\nline:0x390\000\n", '\000' <repeats 196 times>

### 2021-05-10 01:00

耗费1小时28分。

panic问题：

1. sys_printx的参数不正确。
2. 错误字符串为空。
3. CPU不能停止。

调试，我在分析、思考，没有总是盯着代码看。

遗留问题，spin不能阻止进程执行下面的代码。能，是我看错了。那是其他进程打印到这个屏幕的数据。

没开空调，非常非常热。

### 2021-05-10 08:37

目前的中断重入处理机制，在系统调用sys_call使用的函数比如sys_printx中如果出现死循环，将导致进程总是运行触发系统调用的进程。

验证了这点。

在sys_printx中加入死循环，然后在clock_handler设置断点。每次在clock_handler断点都会发现，proc_ready_table是TestA进程。

耗费了30分钟。

如此，时钟中断的意义是什么？若内核中存在死循环，目前的时钟中断处理程序却不能处理他。

### 2021-05-10 10:56

创建发送消息的进程队列，知识盲点，不知道怎么创建链表中的结点。

耗费45分。炒冷饭。忘记了知识点，直接翻看笔记，能节约30多分钟。非必要不自己写代码验证。

自己写简单代码测试，最终查看以前写的代码，在里面找到了例子。

1. 通过测试的代码，要保存好。
2. 去年写过不少CPP代码，已经全部忘记了。
3. vim + tags + Taglist 怎么打开多个文件的tags列表？

### 2021-05-10 12:07

ipc函数--结构体定义、函数声明。

唯一难点，在sys_call中怎么处理拥有不同形参列表的函数。

耗费53分。比较正常。

### 2021-05-10 15:47

写ipc函数。写得时候很顺畅。

耗费1个小时06分。

呵呵，即使是操作系统开发，也存在不少复制粘贴工作，它们是：

1. 常量定义。
2. 相似度极高的系统调用函数，send_msg、receive_msg。

感受：

1. 两个屏幕很好用。
2. vim搭配上ctags和Taglist后很方便：一个屏幕写代码，一个窗口查找函数或结构体定义；subline Text默认不能查看函数列表。
3. 写代码时，我基本没有思考，看着我前天和昨天写的文档翻译。

### 2021-05-10 17:01

写sys_send_msg和sys_receive_msg。

对错不知，反正写完了。收益于文档，无需思考，思路也没有混乱。

耗费了50分钟。

不顺畅的点：

1. 从队列中移除特定元素。

### 2021-05-10 18:53

检查调试IPC函数，修复了全部错误，无视警告。

犯了一个大错误，在macbook上编译操作系统。幸运的是，并不是完全不能编译。否则，我将会把许多正确的代码改成错误的。

更幸运地是，我发现make clean、ld -Ttext 0x30400不能使用，警觉地看了眼vim的标题栏，发现是macbook上的vim。

一些理解不了的语法，在编译器的帮助下修改了。它们是：

```shell
main.c:1963:15: error: expected expression
                pre->next = {sender, NULL};
```

`*(pre->next) = {sender, NULL};`也不行。改写这样的：

```c
// pre->next = {sender_pid, NULL};
// pre->next = &{sender_pid, 0x0};
pre->next->sender_pid = sender_pid;
pre->next->next = 0;
```

编译无错误了。但不知道正确性如何。把get_ticks用ipc改造后才能测试。

耗费时间43分钟。比较顺畅。

1. 检查代码花费了20分钟。
2. 修改错误用了23分钟。
   1. 有的地方用sender_id，有的地方用sender_pid。在mac上使用sed，无效。试验vim的字符串替换命令。
   2. 其他修改。不涉及逻辑修改。
   3. 我认为，还算正常。

### 2021-05-10 20:34

用IPC改写get_ticks，按照已有文档改写，不难。

时间消耗30分钟。

编译器发现一些小错误：

1. 获取指针成员使用了点号。这很不应该。还改了几次。

运行的时候，出现不明原因错误：

```shell
00014033826i[BIOS  ] Booting from 0000:7c00
00287112033e[CPU0  ] fetch_raw_descriptor: LDT: index (10c7) 218 > limit (f)
00287222295i[CPU0  ] WARNING: HLT instruction with IF=0!
```

### 2021-05-10 22:50

测试使用ipc实现的get_ticks，不是预期效果。查看进程表，发现死锁。

我以为是死锁导致异常。

再查看Proc中的发送进程队列，我以为是在内核中使用Struct Message *header导致代码不能正常运行。

Struct Message *header是很常见的写法，做算法题时，我已经用过很多次。但是，在内核中，header的内存地址是0x0。

想了很多方法，修改了很多次，都不行。

我打算修改成于上神那样的发送队列表示法。

耗费时间1小时46分。很不顺利。

1. 多次修改，修改量大，耗费时间。

### 2021-05-11 00:00

只声明不赋值的指针变量，内存地址是0。

奇怪的问题。

还没有解决。

我做了什么？

1. 想改成和于上神一样的方案，理解他的方案。
2. 发现于上神的代码中，也出现过空指针，我认为，不是空指针的问题。
3. 开始解决空指针问题，出现奇怪的数据。

```shell
(gdb) p proc_table[2].header
$6 = (struct MsgSender *) 0x135610
(gdb) p *proc_table[2].header
$7 = {sender_pid = 1363184, next = 0x1}
(gdb) p *proc_table[3].header
$8 = {sender_pid = 1363184, next = 0x1}
(gdb) p *proc_table[4].header
$9 = {sender_pid = 1363184, next = 0x1}
(gdb) p *proc_table[5].header
$10 = {sender_pid = -268435329, next = 0xf000ff53}
```

sender_pid = 1363184, next = 0x1，为什么出现这么奇怪的数据？

### 2021-05-11 11:35

断点运行了很多次，我确认，出现了死锁。我假设我的安排是不会出现死锁的，但为啥还是出现了？

 

```shell
sys_send_msg (msg=0x136040 <proc_stack+1472>, receiver_pid=1, sender=0x14cd80 <proc_table+288>) at main.c:1970
1970		if(receiver->p_flag == RECEIVING && (receiver->p_receive_from == sender_pid
(gdb) p *sender
$209 = {s_reg = {gs = 57, fs = 15, es = 15, ds = 15, edi = 1269578, esi = 1269453, ebp = 1269788, kernel_esp = 1363376, ebx = 1,
    edx = 10, ecx = 1269824, eax = 3, eip = 206506, cs = 7, eflags = 534, esp = 1269744, ss = 15}, ldt_selector = 88, ldts = {{
      seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 251 '\373',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
      seg_base_middle = 0 '\000', seg_attr1 = 243 '\363', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 2,
  ticks = 7, priority = 8, tty_index = 0, name = '\000' <repeats 19 times>, p_flag = 0, p_msg = 0x0, p_send_to = 0, p_receive_from = 0,
  header = 0x1355fc}
(gdb) p *receiver
$210 = {s_reg = {gs = 57, fs = 13, es = 13, ds = 13, edi = 0, esi = 0, ebp = 1269372, kernel_esp = 1363232, ebx = 2, edx = 0,
    ecx = 1269340, eax = 0, eip = 206506, cs = 5, eflags = 4626, esp = 1269312, ss = 13}, ldt_selector = 80, ldts = {{
      seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 187 '\273',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
      seg_base_middle = 0 '\000', seg_attr1 = 179 '\263', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 1,
  ticks = 15, priority = 15, tty_index = 0, name = '\000' <repeats 19 times>, p_flag = 1, p_msg = 0x135e5c <proc_stack+988>,
  p_send_to = 2, p_receive_from = 0, header = 0x1355fc}
(gdb)
```

![image-20210511113747936](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210511113747936.png)



```shell
(gdb) p *receiver->header
$223 = {sender_pid = 1267244, next = 0x313a1 <sys_write+61>}
```

之前还是正常的，出现死锁前面几步，就不正常了。

非常非常低效。

我一直躺在床上断点调试。

发现的问题有：

1. ipc出现死锁。
2. 打印的数据非常少，似乎有某种限制，甚至出现只打印了字符串的一部分的情况。

### 2021-05-11  16:29

一直在运行内核，做了小修改，就运行。

我在玩黑盒子。

耗费时间1个小时30分，唯一的进展是：只能打印有限字符。

不正常的原因是什么？

### 2021-05-11  17:35

果然，出现了死锁！

![image-20210511173635125](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210511173635125.png)

耗费了26分。这26分钟，有价值、有意义！

都开始写操作系统了，怎么还能玩黑盒子？

### 2021-05-11  19:31

耗费了1小时54分。

好多错误！

做的事情：

1. 打印死锁环。
2. 调试死锁环。
3. 小改动、调试、运行。
4. 调试、运行。

有什么发现？

k_reenter，有问题。一直增加，增加大了75。

进程的ticks等于0时仍然递减。

### 2021-05-11  20:40

耗费了。

发现一个奇怪的线索。

通过block实现进程切换。理解不了。k_reenter的值不正常。

经过几次进程切换后，切换到了B，我认为，是通过block切换进来的。

在B中，k_reenter不等于int的最大值。这是不正确的。

A在执行时，被时钟中断打断过一次，然后？

B是通过block进入的还是通过clock进入的？

即使是通过clock进入，k_reenter也会减去1。

通过block，k_reenter也会自减。

猜测无用，能断点验证就好了。

A遭遇第一次时钟中断，k_reenter = 1，为什么？

1. 在A中，k = int_max。
2. 进入clock，k++，应该等于0。
3. 此时，k=1，说明，在进入clock前，又加了一次，是系统中断调用，write。
4. 不调度进程。



```shell
Reading symbols from ./kernel.bin...done.
(gdb) b write
Breakpoint 1 at 0x32948
(gdb) c
The program is not being run.
(gdb) target remote localhost:1234
Remote debugging using localhost:1234
0x0000fff0 in ?? ()
(gdb) c
Continuing.

Breakpoint 1, 0x00032948 in write ()
(gdb) p k_reenter
$1 = 4294967295
(gdb) c
Continuing.

Breakpoint 1, 0x00032948 in write ()
(gdb) c
Continuing.

Breakpoint 1, 0x00032948 in write ()
(gdb) c
Continuing.

Breakpoint 1, 0x00032948 in write ()
(gdb) c
Continuing.

Breakpoint 1, 0x00032948 in write ()
(gdb) c
Continuing.

Breakpoint 1, 0x00032948 in write ()
(gdb) p k_reenter
$2 = 4294967295
(gdb) b send_rec
Breakpoint 2 at 0x32850: file main.c, line 2231.
(gdb) c
Continuing.

Breakpoint 2, send_rec (function=3, msg=0x136040 <proc_stack+1472>, pid=1) at main.c:2231
2231		switch(function){
(gdb) p k_reenter
$3 = 4294967295
(gdb) c
Continuing.

Breakpoint 1, 0x00032948 in write ()
(gdb) p k_reenter
$4 = 4294967295
(gdb) c
Continuing.

Breakpoint 2, send_rec (function=3, msg=0x136040 <proc_stack+1472>, pid=1) at main.c:2231
2231		switch(function){
(gdb) p k_reenter
$5 = 0
```



在send_rec和write设置断点

1. 先在write设置断点，执行6次。
2. 然后在send_rec设置断点，执行3次3后，k_reenter不正常了。
3. 我可以在执行c6次之后开始查看k的值。

1. 在执行write时，发生clock，但是，仍然从clock中恢复，执行write，k=0，正常。
2. 从write恢复时，k还是等于0。这就不正常了。从系统调用中恢复，k应该是max_int。



<bochs:9> xp /1wx 0xd:0x135654
[bochs]:
0x0000000000135654 <bogus+       0>:	0xffffffff



xp /1wx 0xd:0x135654



xp /1wd 0xd:0x135654

xp /1wu 0xd:0x135654

```shell
<bochs:30> xp /1wd 0xd:0x135654
[bochs]:
0x0000000000135654 <bogus+       0>:	-1
<bochs:31> xp /1wu 0xd:0x135654
[bochs]:
0x0000000000135654 <bogus+       0>:	4294967295
```



b 0x0000000305fe



0x000000030561



clock的iret：b 0x00000003065d

sys的cli k，b 0x0000000305fa

1. 先只设置clock的断点，执行三次后，再设置sys的断点。
2. 执行到sys的断点，查看k的变化

失败了。应该设置iret前面的断点。设置到iret，会执行完iretd。

clock的iret前面：0x000000030659。

又失败，还是设置在cli吧    clock   0x000000030590

sys的cli   0x0000000305fd



0x00000003058c



0x0000000305f2





sys-call



Call   

u 0x0000000305f3





00000000000305eb: (                    ): push ebx                  ; 53

u (0x00000000000305eb-6)

u (0x0000000305fa-)

u (0x0000000305fa-15)

0x0000000305f4

0x0000000305f7

sys的write

```
(0) [0x0000000305fa] 0008:00000000000305fa (unk. ctxt): cli                       ; fa
<bochs:21> xp /1wu 0xd:0x135654
[bochs]:
0x0000000000135654 <bogus+       0>:	0
<bochs:22> s
Next at t=24656943
(0) [0x0000000305fb] 0008:00000000000305fb (unk. ctxt): dec dword ptr ds:0x00135654 ; ff0d54561300
<bochs:23> xp /1wu 0xd:0x135654
[bochs]:
0x0000000000135654 <bogus+       0>:	0

```



k_reenter : clock打断系统调用write时，值为1，恢复时，值为0。这是正常的。

从clock中的断点，跳到sys_call中的断点，k_reenter的值，是1，在cli前，这非常奇怪！



```
(0) [0x0000000305f0] 0008:00000000000305f0 (unk. ctxt): call dword ptr ds:[eax*4+218664] ; ff148528560300
```

### 2021-05-12 00:38

又浪费了4个小时52分。

大部分时间，都在做无用功。

我在做什么？想找出指令的内存地址，然后设置断点查看执行过程。

1. 使用xchg在指定位置断点，获取内存地址。
2. 不使用xchg，使用bochs的b断点，在第步获取的内存地址断点。
3. 可是，取消xchg后，之前获取的内存地址，并不是之前对应的指令。
4. 我反复这样做，反复测试，始终没有得到需要的内存地址。

时间就浪费在这种无脑体力劳动上。

一点点进展：

1. 在bochs中能动态设置断点。可是，汇编代码那么多，不能随意设置断点。
2. 执行系统调用后，发生第一次clock后，恢复到系统调用中时，k_reenter的值是1。这是不正确的。但不知道是哪里出错了。
3. 能确定，不是clock有问题。



bochs断点文档

https://blog.csdn.net/yuzhihui_no1/article/details/41446111



找到了正确的查看指令内存地址的方法：

ojbdump -d kernel.bin > dis-k.asm

从第四次时钟中断中恢复，执行TaskSys



执行完N次时钟中断后，ipc就出了问题。

这次时钟中断发生在write过程中，从write中恢复。

断点

```
<bochs:90> info break
Num Type           Disp Enb Address
  1 pbreakpoint    keep y   0x000000031313 							sys_write的第一条指令
  2 pbreakpoint    keep y   0x000000031363 							sys_write的ret
  3 pbreakpoint    keep y   0x000000030561 							hwint0第一条指令，时钟中断
  4 pbreakpoint    keep y   0x0000000316a9 							TaskSys中的call   32eeb <Memset>下条指令
  5 pbreakpoint    keep y   0x0000000305f4 							sys_call-->call函数后的消除参数add    $0xc,%esp
  6 pbreakpoint    keep y   0x000000032849 							sys_receive_msg#ret

```

打印7行字符后，出现问题。

查看 k_reenter

xp /1wu 0xd:0x135654



1. 打印6行字符串后，目前，在 305ed

   1. 这是在 sys_call 的 `call   *0x35628(,%eax,4)`。目前eax是4，根据代码，这个call指令调用 sys_receive_msg。

2. 在这条指令的下一个可能执行的指令设置断点。

   1. Call sys_receive_msg之后，必须执行的指令有哪些？
      1. sys_receive_msg的ret
      2. hwint0的第一条指令
      3. 在sys_receive_msg中，没有其他系统调用，这意味着，整个过程，不会增加k_reenter。

3. 执行过程是：

   1. Sys_receive_msg的第一条指令

   2. Sys_receive_msg的ret

   3. 继续回到sys_call中执行，一直执行到iretd。整个过程中，k_reenter的值正常，到最后，是max_int

   4. 32987：receive_msg的ret

   5. 328b0：send_rec中的call   32978 <receive_msg>后面的参数消除

   6. 后面的的代码是：

      1. ```shell
         328b3:       89 45 f4                mov    %eax,-0xc(%ebp)
         328b6:       eb 11                   jmp    328c9 <send_rec+0x7f>
         328b8:       83 ec 0c                sub    $0xc,%esp
         328bb:       68 a8 2e 03 00          push   $0x32ea8
         328c0:       e8 03 f8 ff ff          call   320c8 <panic>
         328c5:       83 c4 10                add    $0x10,%esp
         328c8:       90                      nop
         328c9:       b8 00 00 00 00          mov    $0x0,%eax
         328ce:       c9                      leave
         328cf:       c3                      ret
         ```

      2. 执行流程会怎样？

         1. 跳转语句、hwint0会改变流程。在跳转语句的下条指令、第一条指令、hwint0设置断点。
         2. 在跳转语句中观察，有没有系统调用。
            1. jmp    328c9 <send_rec+0x7f> 的目标指令是下面那条。
            2. 328c9:       b8 00 00 00 00          mov    $0x0,%eax
            3. 会改变执行流程的，只有 call   320c8 <panic>。这句并没有执行。运行结果，直接跳转到 328c9执行。
         3. k_reenter 是 max_int，正常。

   7. 31200，Printf

      1. 执行流程？
         1. 进入hwint0
         2. 进入sys_call
         3. 究竟是哪种？我在两个函数的第一条指令都设置了断点，运行一下就知道了。
      2. 0x000000031313，进入sys_write，即进入了sys_call。
         1. 此时，k_reenter应该是0。经验证，是的。
      3. 接下来，怎么运行？
         1. 必定可能会执行的指令有：
            1. 31363，sys_write的ret
            2. 0x000000030561，hwint0的第一条指令
      4. 神奇，执行的指令是 `305f4:       83 c4 0c                add    $0xc,%esp`。
         1. 为啥没有执行sys_write的ret？
         2. k_reenter的值变成了1。

   再来一次，精简前面的流程。

   1. 前面6次sys_write都没有问题。重点看第7次sys_write。

   2. 怎么设置断点？

      1. 先在sys_write的第一条指令设置断点，打印出6行字符串。然后再次到达sys_write的第一条指令，逐条观察。

   3. 打印了6行字符串，然后设置断点

      1. ```shell
         <bochs:17> info break
         Num Type           Disp Enb Address
           1 pbreakpoint    keep y   0x000000031313 					sys_write的第一条指令
           2 pbreakpoint    keep y   0x000000030561 					hwint0第一条指令，时钟中断
           3 pbreakpoint    keep y   0x000000030562 					hwint0第二条指令，时钟中断
           0x3063f restore第一条指令
           3 pbreakpoint    keep y   0x000000031363 					sys_write的ret
           
           out_char 的第一条指令 0x31acd
           out_char 的ret   0x31c40
           call out_char 后面的 add   0x31350
           
           
           sys_call 中 call函数后的add消除参数  0x0000000305f4
           
           flush 的第一条指令  0x318d6
           set_cursor 的 ret 0x3196c
           
           1852 31904:  e8 64 00 00 00          call   3196d <set_console_start_video_addr>
         ```

   4. 太神奇了，sys_write不执行它的ret，直接执行call sys_write后面的add 消除参数。

   5. 重要发现：

      1. 可能是out_char这个方法有问题，进入它之后，就进入死循环。



在打印完第6行字符串、打印完第7行字符串的时候，进入out_char就出现了死循环，原因是什么？

只能执行一次。

1. 在sys_write的开头设置断点，执行六行字符串。甚至可以执行7次，如果执行七次后，k_reenter仍然正常。
2. 先验证：打印了6行字符串后，再次执行到sys_write，k是否正常。
3. 在sys_write的开头设置断点，打印6行字符串后，在out_char的第一条指令设置断点。打印完0后，逐条执行。
4. 找到了问题，出现在flush中。
5. 怎么调试？
   1. 先在sys_write的第一条指令设置断点，打印出6行字符串
   2. 在out_char的第一条指令设置断点的最后一个字符串0；
   3. 在flush的第一条指令设置断点，

问题出现在  `31904`，仍然在flush中。



怎么测试？

1. 在sys_write的开头设置断点，执行六行字符串。

2. 先验证：打印了6行字符串后，再次执行到sys_write，k是否正常。

3. 在sys_write的开头设置断点，打印6行字符串后，~~在out_char的第一条指令设置断点。打印完0后，逐条执行。~~

4. 在31904设置断点，然后，看第几次到它时出现问题。假设是第N次。N=9。

5. 重来一次，在第N次达到31904时，逐条观察。

   1. xp /1wu 0xd:0x135654，k=0，正常。

   2. 发生了时钟中断

      1. ```shell
         (0) [0x000000030676] 0008:0000000000030676 (unk. ctxt): ret                       ; c3
         <bochs:40> s
         Next at t=24656526
         (0) [0x0000000319c5] 0008:00000000000319c5 (unk. ctxt): add esp, 0x00000010       ; 83c410
         <bochs:41> s
         Next at t=24656527
         (0) [0x000000030562] 0008:0000000000030562 (unk. ctxt): push ds                   ; 1e
         <bochs:42> s
         Next at t=24656528
         (0) [0x000000030563] 0008:0000000000030563 (unk. ctxt): push es                   ; 06
         <bochs:43> s
         Next at t=24656529
         (0) [0x000000030564] 0008:0000000000030564 (unk. ctxt): push fs                   ; 0fa0
         <bochs:44> s
         Next at t=24656530
         (0) [0x000000030566] 0008:0000000000030566 (unk. ctxt): push gs                   ; 0fa8
         <bochs:45> xp /1wu 0xd:0x135654
         [bochs]:
         0x0000000000135654 <bogus+       0>:	0
         ```

      2. 神奇的是，跳过了pushad#0x30561。

      3. 0x000000030583，xp /1wu 0xd:0x135654，k = 1，正常。

      4. 0x30645，xp /1wu 0xd:0x135654，k = 0，正常。

   3. 从时钟中断中恢复，进入write的ret，0x32957。

      1. 很奇怪，为什么会跳到这里？在0x0000000319c5时发生时钟中断，

         1. ```shell
            319c5:  83 c4 10                add    $0x10,%esp
            319c8:  90                      nop
            319c9:  c9                      leave
            319ca:  c3                      ret
            ```

         2. 应该接着执行这里的指令才合理啊。

         3. 实际执行流程：

            1. Out_char
            2. set_console_start_video_addr
            3. hwint0
            4. write
            5. Printf

         4. 我认为，第4步应该回到set_console_start_video_addr。

怎么办？

1. 打印6行字符串后，在时钟中断设置断点，观察栈中的eip。

2. 仍然没有能在pushad断点，奇怪。

3. 当前，esp是：0x00135558。

   1. 从set_console_start_video_addr进入hwint0，有没有特权级转移？

   2. 执行out_char时，寄存器cs

      1. ```shell
         #out_char
         <bochs:6> sreg
         es:0x000d, dh=0x00cfb300, dl=0x0000ffff, valid=1
         	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
         cs:0x0005, dh=0x00cfbb00, dl=0x0000ffff, valid=1
         	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
         ss:0x000d, dh=0x00cfb300, dl=0x0000ffff, valid=31
         	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
         ds:0x000d, dh=0x00cfb300, dl=0x0000ffff, valid=31
         	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
         fs:0x000d, dh=0x00cfb300, dl=0x0000ffff, valid=1
         	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
         gs:0x0039, dh=0x0000f30b, dl=0x8000ffff, valid=1
         	Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
         ldtr:0x0048, dh=0x00008214, dl=0xcca6000f, valid=1
         tr:0x0040, dh=0x00008b14, dl=0xc380006b, valid=1
         gdtr:base=0x0000000000135660, limit=0x3ff
         idtr:base=0x000000000014c400, limit=0x7ff
         
         ```

      2. ```shell
         # set_console_start_video_addr
         (0) [0x00000003196d] 0005:000000000003196d (unk. ctxt): push ebp                  ; 55
         <bochs:8> s
         Next at t=24582065
         (0) [0x00000003196e] 0005:000000000003196e (unk. ctxt): mov ebp, esp              ; 89e5
         <bochs:9> sreg
         es:0x000d, dh=0x00cfb300, dl=0x0000ffff, valid=1
         	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
         cs:0x0005, dh=0x00cfbb00, dl=0x0000ffff, valid=1
         	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
         ss:0x000d, dh=0x00cfb300, dl=0x0000ffff, valid=31
         	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
         ds:0x000d, dh=0x00cfb300, dl=0x0000ffff, valid=31
         	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
         fs:0x000d, dh=0x00cfb300, dl=0x0000ffff, valid=1
         	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
         gs:0x0039, dh=0x0000f30b, dl=0x8000ffff, valid=1
         	Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
         ldtr:0x0048, dh=0x00008214, dl=0xcca6000f, valid=1
         tr:0x0040, dh=0x00008b14, dl=0xc380006b, valid=1
         gdtr:base=0x0000000000135660, limit=0x3ff
         idtr:base=0x000000000014c400, limit=0x7ff
         ```

   3. 在hwint0时，寄存器cs

      1. ```
         <bochs:14> sreg
         es:0x000f, dh=0x00cff300, dl=0x0000ffff, valid=1
         	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
         cs:0x0008, dh=0x00cf9b00, dl=0x0000ffff, valid=1
         	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
         ss:0x0030, dh=0x00cf9300, dl=0x0000ffff, valid=31
         	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
         ds:0x000f, dh=0x00cff300, dl=0x0000ffff, valid=31
         	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
         fs:0x000f, dh=0x00cff300, dl=0x0000ffff, valid=1
         	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
         gs:0x0039, dh=0x0000f30b, dl=0x8000ffff, valid=1
         	Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
         ldtr:0x0058, dh=0x00008214, dl=0xcdc6000f, valid=1
         tr:0x0040, dh=0x00008b14, dl=0xc380006b, valid=1
         gdtr:base=0x0000000000135660, limit=0x3ff
         idtr:base=0x000000000014c400, limit=0x7ff
         
         ```

      2. cs:0x0008。0特权级。

4. 中断发生时的堆栈

   1. ![image-20210512114956434](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210512114956434.png)

1. xp /1wx 0x00135558
2. xp /1wx 0x00135554
3. xp /1wx (0x00135558+4)
4. xp /1wx (0x00135558+8)
5. xp /1wx (0x00135558+12)
6. xp /1wx (0x00135558+16)
7. xp /1wx (0x00135558+20)

```shell
<bochs:34> xp /1wx (0x00135558+4)
[bochs]:
0x000000000013555c <bogus+       0>:	0x0014cd80
<bochs:35> xp /1wx (0x00135558+8)
[bochs]:
0x0000000000135560 <bogus+       0>:	0x0013559c
<bochs:36> xp /1wx (0x00135558+12)
[bochs]:
0x0000000000135564 <bogus+       0>:	0x00135578
<bochs:37> xp /1wx (0x00135558+16)
[bochs]:
0x0000000000135568 <bogus+       0>:	0x00000009
<bochs:38> xp /1wx 0x9:0x135578
[bochs]:
0x0000000000135578 <bogus+       0>:	0x000319c5
<bochs:39> 
```

进入hwint0后，esp+16才是cs，前面的分别是什么？能确定，esp+12是eip。

Cs:eip对应的指令是, xp /1wx 0x9:0x135578--->0x000319c5。

```shell
# set_console_start_video_addr
319c0:  e8 a2 ec ff ff          call   30667 <out_byte>
319c5:  83 c4 10                add    $0x10,%esp
```



1. 时钟中断的iretd之前，eps：0x0014cdb0
2. xp /1wx 0x0014cdb0		32957
3. xp /1wx (0x0014cdb0+4)        0x00000007
4. xp /1wx (0x0014cdb0+8)	    0x00000212
5. xp /1wx (0x0014cdb0+12)     0x00135f20
6. xp /1wx (0x0014cdb0+16)     0x0000000f





00000000_0014cc70

rsp: 00000000_0014cc60

相差16个字节，分别是：gs、fs、es、ds中的数据占用的空间。

xp /1wx (0x0014cc60+48)

xp /1wx (0x0014cc60+52)

xp /1wx (0x0014cc60+56)



```shell
<bochs:22> xp /1wx 0x5:0x31643
[bochs]:
0x0000000000031643 <bogus+       0>:	0x83e58955
# 0x83e58955 这种排列方式，是小端法。高地址在左边，低地址在右边，和我们常见的数字排列顺序一致。
```

![image-20210512134832263](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210512134832263.png)



`xp /1wx 0x5:0x31643`。这是从时钟中断中返回前，从esp中获取的cs和ip，查看它们的数据，确实是TaskTTY中的指令。这说明，时钟中断，这个时候是正确的。

再看一次，在ipc中出错的时钟中断。

1. 先在sys_write的开头设置断点，打印6行字符串。
2. 然后在hwint0的第二条指令设置断点，进入。

时钟中断开头，执行pushad后，

Esp：00000000_00135558

在执行pop gs前，

esp：00000000_0014cd80



时钟中断开头，存储在堆栈中的cs:eip，是内存地址

```shell
<bochs:19> xp /1wx 0x9:0x135578
[bochs]:
0x0000000000135578 <bogus+       0>:	0x000319c5
```



xp /1wx (0x0014cd80+48)

xp /1wx (0x0014cd80+52)

xp /1wx (0x0014cd80+56)

```shell
<bochs:19> xp /1wx 0x9:0x135578
[bochs]:
0x0000000000135578 <bogus+       0>:	0x000319c5
<bochs:20> xp /1wx (0x0014cd80+48)
[bochs]:
0x000000000014cdb0 <bogus+       0>:	0x00032957
<bochs:21> xp /1wx (0x0014cd80+52)
[bochs]:
0x000000000014cdb4 <bogus+       0>:	0x00000007
<bochs:22> xp /1wx (0x0014cd80+56)
[bochs]:
0x000000000014cdb8 <bogus+       0>:	0x00000212
```

xp 0x00000007:0x00032957



再看一次，正常的时钟中断

执行pushad后

rsp: 00000000_0014cc70

xp /1wx (0x0014cc70 + 0)

xp /1wx (0x0014cc70 + 32)

xp /1wx (0x0014cc70 + 36)

xp /1wx (0x0014cc70 + 40)

xp /1wx (0x0014cc70 + 44)



Cs:eip

<bochs:10>  xp /1wx 0x5:0x00031643
[bochs]:
0x0000000000031643 <bogus+       0>:	0x83e58955
<bochs:11> 



时钟中断，执行pop gs前，

00000000_0014cdb0

xp /1wx (0x0014cc90+0)

xp /1wx (0x0014cc90+4)

xp /1wx (0x0014cc90+8)

xp /1wx (0x0014cc90+12)

<bochs:24> xp /1wx 0x5:0x00031643
[bochs]:
0x0000000000031643 <bogus+       0>:	0x83e58955

耗费了2个小时22分。

确定：

1. 在打印6行IPC结果字符串后，打印完第7行后，时钟中断出现故障。
   1. 发生时钟中断时入栈的cs、eip和iretd时出栈的cs、eip不同。

为什么耗费这么多时间？

1. 多次、反复断点调试。这耗费了大部分时间。
2. TSS知识，出现混乱，把它想明白，用了点时间。
3. 故障cs:eip指向未知错误机器码，我在内核机器码中反复查找，浪费了很多时间。
4. 验证正确的时钟中断，却弄成了验证错误的时钟中断。这是属于思路不清晰。

现在，焦点应该集中于解决：时钟中断为什么出现故障？

一、故障时，在clock_handler、schedule_process中是否发生了切换。

1. 打印6行字符串，在schedule_process、clock_handler设置断点。

1. ```shell
   <bochs:17> info break
   Num Type           Disp Enb Address
     1 pbreakpoint    keep y   0x000000031313 					sys_write的第一条指令
     2 pbreakpoint    keep y   0x000000030561 					hwint0第一条指令，时钟中断
     3 pbreakpoint    keep y   0x000000030562 					hwint0第二条指令，时钟中断
     0x3063f restore第一条指令
     3 pbreakpoint    keep y   0x000000031363 					sys_write的ret
     
     out_char 的第一条指令 0x31acd
     out_char 的ret   0x31c40
     call out_char 后面的 add   0x31350
     out_byte 的ret						0x30676
     
     
     sys_call 中 call函数后的add消除参数  0x0000000305f4
     
     flush 的第一条指令  0x318d6
     set_console_start_video_addr 的 ret 0x3196c
     
     clock_handler 的第一条指令	0x313a0
     schedule_process  的第一条指令	0x3127b
     proc_ready_table 的内存地址  0x135a60
     
     
     
     1852 31904:  e8 64 00 00 00          call   3196d <set_console_start_video_addr>
   ```



查看结果

rsp: 00000000_0014cdb0

xp /1wx (0x0014cdb0+0)

xp /1wx (0x0014cdb0+4)

xp /1wx (0x0014cdb0+8)



堆栈，没有切换，但是，eip和cs指向的指令不正确。

二、在打印6行字符串后，在0x319c5后发生时钟中断

1. 验证方法，在sys_write第一条指令设置断点，打印完6行字符串后，在0x319c5设置断点，一直打印完打印完第7行字符串。
2. 注意，从0开始，要观察数据。
   1. 0x319c5究竟有没有执行？
   2. 如果没有执行，eip就应该是0x319c5；如果执行了，eip就应该是0x319c8。
   3. 当然，要先验证，是否在0x319c5发送了时钟中断。
   4. 在 0x319c5 设置断点，打印完第七行的0字符串后，执行第二次c时，卡住了。

```shell
<bochs:19> r
CPU0:
rax: 00000000_00001555
rbx: 00000000_00000009
rcx: 00000000_0000173d
rdx: 00000000_000003d5
rsp: 00000000_00135584
rbp: 00000000_0013559c
rsi: 00000000_0014cd80
rdi: 00000000_00135f43
r8 : 00000000_00000000
r9 : 00000000_00000000
r10: 00000000_00000000
r11: 00000000_00000000
r12: 00000000_00000000
r13: 00000000_00000000
r14: 00000000_00000000
r15: 00000000_00000000
rip: 00000000_000319c5
eflags 0x00000246: id vip vif ac vm rf nt IOPL=0 of df IF tf sf ZF af PF cf

<bochs:20> sreg
es:0x000f, dh=0x00cff300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
cs:0x0008, dh=0x00cf9b00, dl=0x0000ffff, valid=1
	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
ss:0x0030, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
ds:0x000f, dh=0x00cff300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
fs:0x000f, dh=0x00cff300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
gs:0x0039, dh=0x0000f30b, dl=0x8000ffff, valid=1
	Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0058, dh=0x00008214, dl=0xcdc6000f, valid=1
tr:0x0040, dh=0x00008b14, dl=0xc380006b, valid=1
gdtr:base=0x0000000000135660, limit=0x3ff
idtr:base=0x000000000014c400, limit=0x7ff

```

c是指在当前语句未执行前停住，点击s，执行的就是当前被卡住的这句。

执行之后，到了哪里？

从时钟中断中返回，eip应该是：31350。

上面的调试方法能再优化：

1. 先打印6行，在 0x319c5 设置断点，打印完第七行的0字符串后，在 0x31348 设置断点。

print-stack





0x000000031313 					sys_write的第一条指令



断点

0x3196d

第7行，第10次断点会被卡住，在第9次c逐条查看，中途被卡住。

接着上一步，在 0x319c0 中间会被卡住，在里面，逐条查看，验证方法是：

1. 在 0x000000031313 设置断点，打印6行字符串。
2. 在 0x3196d 设置断点，在第9次c后逐条查看
3. 在0x319c0设置断点，进入里面逐条查看。

![image-20210512194150409](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210512194150409.png)

执行完30676后，发生时钟中断。

此时，k_reenter = 0，proc_ready_table的值是 0x0014cd80。

eip是什么？

0x319c0的下一条指令：319c5。

检查堆栈中，是不是这个。

在下文的print-stack打印出的堆栈、也就是当前rsp中的eip是30676。

这意味着，在30675点击s时，已经进入了时钟中断。

tss中的esp0

0x0030:0x0014cdc4

tss中的esp0 和 "proc_ready_table的值是 0x0014cd80"的关系是什么？和当前的rsp 00000000_00135554 又是什么关系？

esp0和proc_ready_table可能是一样东西，sp与二者相差太多。

一、sp是否正确？

1. 验证方法，观察正确的时钟中断发生时，sp是否与前两个接近。
2. 如果我不正确，我能怎么办？时钟中断发生时，选择哪个中断，是CPU自动确定的。目前的sp是哪里来的？



```shell
<bochs:39> print-stack
Stack address size 4
				 内存地址			对应地址中的数据
 | STACK 0x00135554 [0x00135f43] (<unknown>)
 | STACK 0x00135558 [0x0014cd80] (<unknown>)
 | STACK 0x0013555c [0x0013559c] (<unknown>)
 | STACK 0x00135560 [0x00135574] (<unknown>)
 | STACK 0x00135564 [0x00000009] (<unknown>)
 | STACK 0x00135568 [0x000003d5] (<unknown>)
 | STACK 0x0013556c [0x00001555] (<unknown>)
 | STACK 0x00135570 [0x00001555] (<unknown>)
 | STACK 0x00135574 [0x00030676] (<unknown>)
 | STACK 0x00135578 [0x00000008] (<unknown>)
 | STACK 0x0013557c [0x00000246] (<unknown>)
 | STACK 0x00135580 [0x000319c5] (<unknown>)
 | STACK 0x00135584 [0x000003d5] (<unknown>)
 | STACK 0x00135588 [0x00001555] (<unknown>)
 | STACK 0x0013558c [0x00000000] (<unknown>)
 | STACK 0x00135590 [0x00000000] (<unknown>)
<bochs:40> xp /1wx 0x0030:0x135f43
[bochs]:
0x0000000000135f43 <bogus+       0>:	0x00000a00
<bochs:41> xp /1wx 0x30:0x135554
[bochs]:
0x0000000000135554 <bogus+       0>:	0x00135f43
<bochs:42> xp /1wx 0x13558c
[bochs]:
0x000000000013558c <bogus+       0>:	0x00000000
<bochs:43> xp /1wx 0x135570
[bochs]:
0x0000000000135570 <bogus+       0>:	0x00001555
<bochs:44> 
```



在hwint0中，在clock_handler之前，print-stack打印出来的堆栈，就是CPU正在使用的堆栈，也就是通过r查看到的rsp。

那么，疑问有：

1. tss中的esp0是什么？
2. proc_ready_table，又是啥？是当前正在执行却被挂起的进程的进程表。

在print-stack（即r查看到的rsp）中，查询eip，逻辑上是合理的。

xp /1wx (0x00135544)

eip:   xp /1wx (0x00135544+48)

Cs: xp /1wx (0x00135544+52)

pid    xp /1wx (0x00135544 + 68 + 32)



proc_ready_table

xp  /1wx  0x135a60		data	0x0014cd80

xp /1wx (0x0014cd80)

eip:   xp /1wx (0x0014cd80+48)

Cs: xp /1wx (0x0014cd80+52)

xp /1wx (0x0014cd80+68+32)

```
<bochs:69> xp /1wx (0x0014cd80+68+32)
[bochs]:
0x000000000014cde4 <bogus+       0>:	0x00000001
```

当前，是第2个进程，TaskSys。

从时钟中断恢复时，执行write的ret指令。假设它是正确的，怎么解释？

没意义。解释错误的东西无意义。

时钟中断打断的是out_byte，如果中途没有切换进程，就应该回到out_byte。

1. A进程通过系统调用执行write--->sys_call--->sys_write--->out_char
2. 发生时钟中断时，保存的应该是A进程的快照。



k

xp /1wu 0x135654



### 一个正常的时钟中断

```shell
(0) [0x000000030562] 0008:0000000000030562 (unk. ctxt): push ds                   ; 1e
<bochs:4> r
CPU0:
rax: 00000000_00000000
rbx: 00000000_00000000
rcx: 00000000_00000000
rdx: 00000000_00000000
rsp: 00000000_0014cc70
rbp: 00000000_00000000
rsi: 00000000_00000000
rdi: 00000000_00000000
r8 : 00000000_00000000
r9 : 00000000_00000000
r10: 00000000_00000000
r11: 00000000_00000000
r12: 00000000_00000000
r13: 00000000_00000000
r14: 00000000_00000000
r15: 00000000_00000000
rip: 00000000_00030562
eflags 0x00001002: id vip vif ac vm rf nt IOPL=1 of df if tf sf zf af pf cf
<bochs:5> info tss
tr:s=0x40, base=0x000000000014c380, valid=1
ss:esp(0): 0x0030:0x0014cca4
ss:esp(1): 0x0000:0x00000000
ss:esp(2): 0x0000:0x00000000
cr3: 0x00000000
eip: 0x00000000
eflags: 0x00000000
cs: 0x0000 ds: 0x0000 ss: 0x0000
es: 0x0000 fs: 0x0000 gs: 0x0000
eax: 0x00000000  ebx: 0x00000000  ecx: 0x00000000  edx: 0x00000000
esi: 0x00000000  edi: 0x00000000  ebp: 0x00000000  esp: 0x00000000
ldt: 0x0000
i/o map: 0x0000
<bochs:6> xp /1wx xp  /1wx  0x135a60
:6: syntax error at 'xp'
<bochs:7> xp  /1wx  0x135a60
[bochs]:
0x0000000000135a60 <bogus+       0>:	0x0014cc60

```

A.rsp是：00000000_0014cc70

B.tss中，ss:esp(0): 0x0030:0x0014cca4，

C.proc_ready_table：0x0014cc60

1. B是寄存器堆栈的最高地址。
2. A从最高地址开始，入栈了cs、eip、eflags、esp、ss和pushad共13个元素，共13*4=52个字节。
3. 0x0014cca4 - 0x0x0014cc70 = 0x3 + 0x4 = 52。
4. proc_ready_table指向寄存器栈Regs的最低地址，和最高地址的差距是sizeof(Regs) = 0x4 + 4 = 68个字节。

这是正确的。



Cs:eip

cs:xp /1wx (0x0014cc70+32)

Ip:xp /1wx (0x0014cc70+36)

xp /1wx (0x0014cc70 + 68 + 32)



### 收获

1. 大量断点、单步调试捕捉出问题的操作，耗费特别特别多时间。体力劳动。
2. 在错误的时钟中断现象，没根据地猜测了很长时间。
3. 笔记非常混乱了，不整理好，这几天的心血就全部浪费。
4. 确认，问题出现在时钟中断时选择的esp是错误的。选择了内核栈吗？



cs:xp /1wx (0x0084ce10+48)

Ip:xp /1wx (0x0084ce10+52)

xp /1wx (0x0084ce10 + 68 + 32)



### 2021-05-13 03：25

耗费了4个小时20分。

我做了什么？

1. 断点调试。分别用gdb和bochs，仍然没有找到错误原因。束手无策了。
2. 不能再继续这种状态了。一直像这样无脑单步执行，解决不了任务问题，还在熬夜！



0x40--->0100 0000

01000

第8个

0x0000000000835660 + 0x40

0x00000000008356a0

```
<bochs:10> info tss
tr:s=0x40, base=0x000000000084c380, valid=1
ss:esp(0): 0x0030:0x0084cdc4
ss:esp(1): 0x0000:0x00000000
ss:esp(2): 0x0000:0x00000000
cr3: 0x00000000
eip: 0x00000000
eflags: 0x00000000
cs: 0x0000 ds: 0x0000 ss: 0x0000
es: 0x0000 fs: 0x0000 gs: 0x0000
eax: 0x00000000  ebx: 0x00000000  ecx: 0x00000000  edx: 0x00000000
esi: 0x00000000  edi: 0x00000000  ebp: 0x00000000  esp: 0x00000000
ldt: 0x0000
i/o map: 0x0000
<bochs:11> sreg
es:0x000f, dh=0x00cff300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
cs:0x0007, dh=0x00cffb00, dl=0x0000ffff, valid=1
	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
ss:0x000f, dh=0x00cff300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
ds:0x000f, dh=0x00cff300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
fs:0x000f, dh=0x00cff300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
gs:0x0039, dh=0x0000f30b, dl=0x8000ffff, valid=1
	Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0058, dh=0x00008284, dl=0xcdc6000f, valid=1
tr:0x0040, dh=0x00008b84, dl=0xc380006b, valid=1
gdtr:base=0x0000000000835660, limit=0x3ff
idtr:base=0x000000000084c400, limit=0x7ff
<bochs:12> xp /1wx (0x000000000084c380+4)
[bochs]:
0x000000000084c384 <bogus+       0>:	0x0084cdc4

```



84c380



在每次中断的开头，检查CPU选择 堆栈和TSS中的esp0是否一致。

[tss+4]---->esp0

esp-----> 在push gs后，二者相差68



```shell
<bochs:6> xp /1wx (0x84cce4-8)
[bochs]:
0x000000000084ccdc <bogus+       0>:	0x00835cc0
<bochs:7> xp /1wx (0x84cce4-12)
[bochs]:
0x000000000084ccd8 <bogus+       0>:	0x00001202
<bochs:8> xp /1wx (0x84cce4-16)		#cs
[bochs]:
0x000000000084ccd4 <bogus+       0>:	0x00000005
<bochs:9> xp /1wx (0x84cce4-20)		#eip
[bochs]:
0x000000000084ccd0 <bogus+       0>:	0x00031724

```



### 2021-05-13 18:44

调试方式升级了，我用汇编代码在各种中断中捕捉建立快照的esp和tss.esp0不一致的情况，比不断单点执行的愚蠢方式先进几千倍吧。

根据数据，我认为：

1. 在时钟中断中建立快照的时候，选择了内核堆栈，而不是选择tss.esp0。
2. 从时钟中断中恢复时，选择了proc_ready_table，而，proc_ready_table和tss.esp0是一致的。

究竟是什么原因导致从系统调用中进入时钟中断时CPU没有选择tss.esp0而是选择了内核栈？

从系统调用进入时钟中断时会如此。

也就是说，发送中断重入时会如此。

84cce4

```shell
(gdb) p &proc_table[0]
$2 = (Proc *) 0x84cca0 <proc_table>
(gdb) p &proc_table[1]
$3 = (Proc *) 0x84cd30 <proc_table+144>
(gdb) p &proc_table[2]
$4 = (Proc *) 0x84cdc0 <proc_table+288>
(gdb) p &proc_table[3]
$5 = (Proc *) 0x84ce50 <proc_table+432>
(gdb) info tss
Undefined info command: "tss".  Try "help info".
(gdb) p tss
$6 = {last_tss_ptr = 0, esp0 = 8703204, ss0 = 48, esp1 = 0, ss1 = 0, esp2 = 0, ss2 = 0, cr3 = 0, eip = 0, eflags = 0,
  eax = 0, ecx = 0, edx = 0, ebx = 0, esp = 0, ebp = 0, esi = 0, edi = 0, es = 0, cs = 0, ss = 0, ds = 0, fs = 0,
  gs = 0, ldt = 0, trace = 0, iobase = 108}
```

### 2021-05-14 09:10 

唉！！！！！

耗时几天的IPC诡异问题，在今天早晨花了2个多小时解决了。

怎么解决的？碰运气浏览《操作系统真相还原》，读到这么一个知识点：从非0特权级到0特权级时，CPU才会选择tss.esp0；从0特权级到0特权，并不使用tss.esp0，而是直接使用内核栈。

昨天看到，在中断重入时，建立快照的esp和tss.esp0不一致。看到这个知识点，我豁然开朗了！

再看于上神的代码，人家就是那么处理的。我看了他的代码，却熟视无睹。我以为我的代码和他的代码只是形式上的差异。实际上，却是重大的逻辑差异。

细小的判断，不熟悉汇编指令的跳转指令，让我耗费了半个多小时。还有细小错误。

bochs的错误信息，基本上没有用处，或者，我不熟悉这些报错信息。

在搜索引擎搜索这些信息，没有啥参考资料。这不是热门问题。

像其他语言，搜一下报错信息，能找到执行一下就能解决问题的方案。

不过，仍然很高兴。又可以往前推进操作系统的开发进度了。

可是，又遇到问题了。

1. 之前的IPC死锁问题。
2. fetch_raw_descriptor: GDT: index (d517) 1aa2 > limit (3ff)

### 2021-05-14 09:25

fetch_raw_descriptor: GDT: index (d517) 1aa2 > limit (3ff)

解决了！花了几分钟。

碰运气，看kernel.asm代码，发现，hwint0中建立快照的代码，少了`push fs push gs`。堆栈是错误的，寄存器中的值相应也会是错的。

### 2021-05-14 12:23

死锁频繁发生，是因为我在get_ticks_pic（获取ticks的函数）中把source写成固定值。

耗费时间最多的是单步执行。没有发挥作用。

现在，还有一些小问题。原因不明。

耗费1个小时12分。还行。

### 2021-05-14 16:45

写itoa函数。耗费了3小时11分。仍没有写出一个健壮的函数，转化大于10的整型数就会导致我的操作系统崩溃。

这个函数可不简单，包含了不少内容。

1. 怎么用函数返回处理之后的字符串。
2. `*(*p)++`怎么理解？
3. 字符串的结尾字符必须是'\0'。
4. 在我的操作系统中，内核堆栈和全局变量、局部变量都有非常大的关系。局部变量和进程的堆栈确定无疑有关系。
5. 段错误。

为什么花了这么长时间？

1. 我以为第2步中的语句是正确的。印象中，我好像看到过类似的语句。为了写出正确的语句，我不断尝试各种写法，现在看来，都是同质的。后来，上网查”段错误“，发现，这是非常非常常见的错误写法。

2. ```c
   char *str;
   *str++ = 'A';
   *str++ = 'B'
   ```

3. 第2步，耗费了主要时间。我不应该一直运行各种错误的同质写法，而应该早点查资料。

4. `*(*p)++)`和`**p++`是不同的。要把字符串的最后一个字符赋值为空字符，正确的写法是`*(*p) = '\0'`。

   1. 我并不知道这些形式差异不大的所有写法的含义，没记住。
   2. 需要的时候，测试一下吧。
   3. C语言的变量的作用范围很严格。在switch的case中声明局部变量要用花括号把这个case括起来。



又遇到问题，不知道是什么原因。

```shell
00025324359e[CPU0  ] fetch_raw_descriptor: LDT: index (d157) 1a2a > limit (f)
00025434618i[CPU0  ] WARNING: HLT instruction with IF=0!
```

是itoa造成的，仍然不能转换大于10的数字。

还有一个问题，打印一些时钟中断后，就一直停在了tty_to_read。

直接原因是：

```shell
(gdb) p proc_table[0]
$1 = {s_reg = {gs = 57, fs = 13, es = 13, ds = 13, edi = 0, esi = 0, ebp = 4415596, kernel_esp = 4509872, ebx = 0,
    edx = 4497664, ecx = 123, eax = 4497888, eip = 204166, cs = 5, eflags = 4630, esp = 4415596, ss = 13},
  ldt_selector = 72, ldts = {{seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000',
      seg_attr1 = 187 '\273', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {
      seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 179 '\263',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 0, ticks = 1, priority = 2,
  tty_index = 1, name = '\000' <repeats 19 times>, p_flag = 0, p_msg = 0x0, p_send_to = 0, p_receive_from = 0,
  header = 0x0}
(gdb) p proc_table[1]
$2 = {s_reg = {gs = 57, fs = 13, es = 13, ds = 13, edi = 0, esi = 0, ebp = 4416156, kernel_esp = 4510016, ebx = 4,
    edx = 0, ecx = 4416124, eax = 0, eip = 207751, cs = 5, eflags = 4630, esp = 4416096, ss = 13}, ldt_selector = 80,
  ldts = {{seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 187 '\273',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
      seg_base_middle = 0 '\000', seg_attr1 = 179 '\263', seg_limit_high_and_attr2 = 207 '\317',
      seg_base_high = 0 '\000'}}, pid = 1, ticks = 2, priority = 2, tty_index = 1, name = '\000' <repeats 19 times>,
  p_flag = 1, p_msg = 0x43627c <proc_stack+988>, p_send_to = 4, p_receive_from = 0, header = 0x435618}
(gdb) p proc_table[2]
$3 = {s_reg = {gs = 57, fs = 15, es = 15, ds = 15, edi = 4416351, esi = 4416232, ebp = 4416572, kernel_esp = 4510160,
    ebx = 1, edx = 68, ecx = 4416608, eax = 0, eip = 207751, cs = 7, eflags = 530, esp = 4416528, ss = 15},
  ldt_selector = 88, ldts = {{seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000',
      seg_attr1 = 251 '\373', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {
      seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 243 '\363',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 2, ticks = 1, priority = 1,
  tty_index = 0, name = '\000' <repeats 19 times>, p_flag = 1, p_msg = 0x436460 <proc_stack+1472>, p_send_to = 1,
  p_receive_from = 0, header = 0x0}
(gdb) p proc_table[3]
$4 = {s_reg = {gs = 57, fs = 15, es = 15, ds = 15, edi = 4416867, esi = 4416744, ebp = 4417084, kernel_esp = 4510304,
    ebx = 1, edx = 10, ecx = 4417120, eax = 0, eip = 207751, cs = 7, eflags = 530, esp = 4417040, ss = 15},
  ldt_selector = 96, ldts = {{seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000',
      seg_attr1 = 251 '\373', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {
      seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 243 '\363',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 3, ticks = 1, priority = 1,
  tty_index = 1, name = '\000' <repeats 19 times>, p_flag = 1, p_msg = 0x436660 <proc_stack+1984>, p_send_to = 1,
  p_receive_from = 0, header = 0x0}
(gdb) p proc_table[4]
$5 = {s_reg = {gs = 57, fs = 15, es = 15, ds = 15, edi = 4417385, esi = 4417256, ebp = 4417596, kernel_esp = 4510448,
    ebx = 1, edx = 70, ecx = 4417632, eax = 0, eip = 207767, cs = 7, eflags = 530, esp = 4417552, ss = 15},
  ldt_selector = 104, ldts = {{seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000',
      seg_attr1 = 251 '\373', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {
      seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 243 '\363',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 4, ticks = 1, priority = 1,
  tty_index = 2, name = '\000' <repeats 19 times>, p_flag = 2, p_msg = 0x436860 <proc_stack+2496>, p_send_to = 0,
  p_receive_from = 1, header = 0x435618}
```

### 2021-05-14 18:39

IPC、用IPC改造后的get_ticks，基本能认为是正常的了。但仍存在奇怪的问题。

1. A进程总是只能打印少数几条数据。B、C打印的数据条数是正常的。
2. Printf的%d有问题。
3. 内核栈大小对程序运行有影响。

我做了什么？小改动；反复运行、测试。

耗费了39分钟。比较低效。不知道还有哪些问题。

### 2021-05-14 19:01

assert等条数函数出问题了，出现Invalid opcode，原因未知。

我做了什么？一直测试运行，玩黑盒子。耗费了19分。

### 2021-05-14 19:52

assert等条数函数出问题了，出现Invalid opcode，原因未知。

找到了直接原因，是因为

```shell
void assertion_failure(char *exp, char *filename, char *base_filename, unsigned int line)
{
	printx("%c%s error in file [%s],base_file [%]s,line [%d]\n\n",
		ASSERT_MAGIC, exp, filename, base_filename, line);
	spin("Stop Here!\n");
}
```

造成的。换成下面的字符串，就没有问题。

```c
void assertion_failure(char *exp, char *filename, char *base_filename, unsigned int line)
{
			Printf("t = %s\n", "Hello");
}
```

```c
Printf("%c%s error in file [%s],base_file [%]s,line [%d]\n\n",38, "===========", "===========", "============", 2);
```

这个打印函数，存在极大 的问题。例如，像上面这种很常见的字符串和格式，都出现奇怪的结果。

我怎么在调试？

1. assert导致问题，查看它的整个过程：调用的函数。

2. 单步执行追踪，跟踪完了它调用的所有函数，没有在函数执行过程中出问题。

3. 再往后面执行，很多指令执行之后，还是没问题，我不想浪费时间，点击'c'往下执行，出现了问题。

4. 但是，不知道问题出现在哪一句。

5. 再试一次，更耐心地追踪到出问题的语句S，是没有增加assert前能正常运行的语句。

6. 我认为，不是S导致的问题。在这里，千万不能误入歧途。

7. 我放弃了单步追踪问题的思路，费时，还发现不了有用的线索。执行着，突然，在总是正常的语句那里出现Invalid Code。

8. 我修改了assertion_failure中打印的字符串，设置了一个很简单的字符串，不出现问题。

9. 根据这一点，我认为，是打印字符串的函数出了问题。最大怀疑对象是vsprintf。

10. vsprintf会导致系统崩溃。

11. 小修改，然后测试，

    1. ```c
       // [%]s会导致打印出奇怪的未知字符
       printx("%c%s error in file [%s],base_file [%]s,line [%d]\n\n",
       		ASSERT_MAGIC, exp, filename, base_filename, line);
       ```

    2. ```c
       // 把line换成5，不会出现问题。
       printx("%c%s error in file [%s],base_file [%]s,line [%d]\n\n",
       		ASSERT_MAGIC, exp, filename, base_filename, 5);
       ```

    3. 我猜测，是vsprintf中的%d出了问题。

12. 测试Printf("%d", 389)。

13. Itoa，单独测试转换389，没有问题。

```shell
receive_msg		0x32bd8
```



两次断点执行到 32bd8后，再执行到32a3c，然后逐条执行。

```
32a74:       ff 75 0c                pushl  0xc(%ebp)
   ~                          |3286    32a77:       e8 5c 01 00 00          call   32bd8 <receive_msg>
```



用gdb不能查看到究竟在哪里出错了，用bochs断点，看清楚了。

```shell
Next at t=25485098
(0) [0x000000032be7] 0005:0000000000032be7 (unk. ctxt): ret                       ; c3
<bochs:31> s
Next at t=25485099
(0) [0x000000000005] 0005:0000000000000005 (unk. ctxt): inc dword ptr ds:[eax]    ; ff00
```

汇编函数receive_msg从sys_call回归后，ret的值变得异常。为啥会这样？放在内核栈中的eip，被某个操作修改了。

递归函数，会破坏堆栈。

这个错误，是常见的错误，有专业叫法”堆栈溢出“。常见于汇编。

于上神的代码为啥没有这个问题。我的inner_buf和他的一样大小，内核栈比他大许多。

### 2021-05-15 00:23

解决了上面的递归导致堆栈溢出的问题。

为啥能解决？

对比于上神的代码，重看我的代码。我发现，

1. 我的进程堆栈只有128字节，是在进程初始化时写的固定值。
2. 堆栈的初始化不正规。是从低地址向高地址方向。

最大时间消耗在哪里？

定位出错原因是堆栈---》是进程堆栈，而不是和操作系统的内存分布有关。

我写的固定值在进程初始中，无论怎么修改定义的堆栈常量，都是无用的。

### 2021-05-15 09:56

3个用户进程，第3个用户进程C进程一个打印字符也不打印。我设置了，预期应该打印数个字符。

怎么调试？

1. 用gdb调试，不设置断点。
2. 当操作系统运行至停止时，这时，每个终端都不会打印出新字符。我使用ctrl+c断开程序。
3. 此时，正在TaskTTY中，proc_table中的5个元素，只有第4个元素不正常。p_flag 不是0，p_msg不是0。
4. 在kernel_main设置断点，发现，proc_table[4]在kernel_main的开始和结束，都是不正常。
5. 不清楚为何会如此。不过，我依然在第一次使用proc_table前，使用Memset初始化了它。
6. 用上面的方法再查看proc_table，每个元素的初始值都是正常的。
7. 再次运行，正常了。
8. 不过，仍然有问题：当每个进程中打印字符串的次数太多时，仍然会出现 hlt 问题。原因未知。



IPC功能，到现在，我就打算结束它了。前后耗时，只统计写代码和调试时间，8天。

### 2021-05-15 10:08

用ipc实现的milli_delay函数，会导致系统崩溃。

报错信息是：

```shell
00028168191e[CPU0  ] fetch_raw_descriptor: LDT: index (1297) 252 > limit (f)
00028278447i[CPU0  ] WARNING: HLT instruction with IF=0!
```

不知道从哪里着手去看这个问题。

使用bochs断开，发现下一条要执行的指令是hwint0中的pusad，可是，却卡住在这里，无法往下执行。

是因为，堆栈耗尽了吗？可是，正常使用，堆栈，是会不断被释放的，不应该这么快耗尽啊。

先搁置吧。



### c

```html
为了兼容c中char *p="abc"这种现象的存在，C++特别允许初始化时const char*到char*的自动转换。但是这条规则被 C++ 标明为 “Deprecated”，不被推荐使用。
综上所述，
在 C 中：char *p="abc"是完全合乎规则的事情
在 C++ 中：由于有特殊规定，所以这样也可以。
但在c++中要注意：
char *p="abc" 能不能编译通过要看你使用的编译器。鉴于大量遗留代码的存在，大部分编译器允许其通过，或者给个警告。当然，程序员自己必须保证绝不去修改其值：
a. 程序员不应该在代码中出现*p='A'这样的语句。这是当初约定好了的：编译器允许char *p="abc"通过，而程序员保证不去修改它。 
b. *p='A'编译时应该允许通过，因为单就这条语句而言，它完全合法。 
c. 运行时*p='A'能不能通过要看实际的运行环境，包括你使用的操作系统、编译器、编译器选项 等等，一句话，其运行结果由不得你，且不应该由你去关心，因为这种行为本身已经违反约定了。
```



段错误

https://baike.baidu.com/item/%E6%AE%B5%E9%94%99%E8%AF%AF