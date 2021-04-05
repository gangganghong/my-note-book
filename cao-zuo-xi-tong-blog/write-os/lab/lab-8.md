# 在内核中添加中断处理--简化版本

代码在：/home/cg/os/pegasus-os/v16。

## 无题

不知道拟定一个什么样的小标题，所以干脆就叫”无题“。

我对本实验要做什么非常陌生，第一次看的时候可能跳过这部分内容，那个时候，我总感觉不理解中断，看了这块也没用。

浏览书中的相关部分，好多代码。我觉得这是块硬骨头。

已经在汇编中做完了”中断、时钟中断“实验，现在简单说说中断知识。

就讲讲使用中断的流程吧。

1. 初始化8259A。
2. 建立IDT。
   1. IDT是中断向量表，和GDT一样，也是一块内存。差异是，GDT中的描述符指向各种各样的代码段，IDT中的描述符是门描述符，指向中断例程。
   2. 说得再简单一些，IDT类似PHP中的数组，中断向量号是数组的键，中断向量号对应的中断例程是数组的值。
   3. 给定一个中断向量号，比如"80h"，在IDT中找到对应的中断例程。
3. 建立存储IDT的基址和界限的元结构IdtPtr。
4. 使用`lidt [IdtPtr]`加载IDT。必须在保护模式中加载。
5. 使用`int 中断向量号`触发中断。
6. 时钟中断不需要手工使用`int 中断向量号`触发，而是由另外一个硬件`8253`触发。
7. `8253`是可编程的，能通过它设置时钟中断频率。
   1. 向端口0x43写入时钟中断模式数据。
   2. 向8253的计数器counter0写入计数器的初始值C。
   3. C的单位是秒，C每隔一秒递减一次，在C秒内发生M次时钟中断。
   4. 调整C，时钟中断之间大的时间间隔就会相应调整。
   5. 如果希望每10毫秒发生一次时钟中断，那么，每秒发生100次时钟中断。
   6. M次时钟中断需要的秒数是：R = M / 100。
   7. R就是counter0的值。

## 理论

### 回忆

在loader中已经添加了中断，在内核中又添加中断，那是要做什么？

我猜测，和GDT类似。要在内核中使用中断，需要把loader中的IDT复制到内核中。

想不出有用的东西，直接看书吧，不浪费时间在瞎想上了。

### 学习

#### v1

猜得没错，在内核中添加中断处理，和切换GDT类似。不过，并不是用Memcpy把loader中的IDT复制到内核中。那么，是重新建立了IDT吗？

中断发生时，入栈的数据有eflags、cs、eip、错误码。

需要添加的中断例程非常多，好像有十多个，有除零错误等。为啥是这些而不是其他的？有什么讲究或什么原因吗？是不是和那张中断向量号表有关系？

有讲究，我认为，是一种约定俗成的规范，详情参看：https://docs.huihoo.com/help-pc/int-int_table.html。

上面的中断例程，都是异常例程。

这些异常例程的函数是用汇编写的，结构很相似。声明函数，在函数内调用异常函数，在调用之前会通过压栈传递参数。

异常处理函数会打印出错误码等信息。这是很有用的调试工具。

实现一些函数，打印字符串、打印字符串--设置字符的颜色、把字符串转成整型。

依然用sidt从loader中获取IDT，修改后，再用lidt加载新IDT。

#### v2

##### 修改idt_ptr

声明全局数组变量idt[256]，数组元素是门描述符。

声明全局数组变量idt_ptr[6]，存储IDT的基址和界限。

创建两个指针，分别指向idt_ptr的低2位和高4位。伪代码如下：

```c
// 这两句并不是为了从idt_ptr中获取数据，而是为了创建两个指针指向idt_ptr的低2位和高4位。
short *pm_idt_limit = (short *)&idt_ptr[0];
int *pm_idt_base = (int *)&idt_ptr[2];
// 通过对指针变量赋值修改idt_ptr
// *pm_idt_limit 的值应该是 short 类型。C语言的语法就是这样。
// 如果对pm_idt_limit赋值，值应当是内存地址。
*pm_idt_limit = 256 * sizeof(中断门描述符);
// *pm_idt_base 的值应该是int类型。虽然内存地址也是int类型，但还是应该使用(int)强制类型转化
*pm_idt_base = (int)&idt;

// 往idt中填充中断描述符
// idt是一个全局数组变量。在它创建之后，它就具有内存地址。
// 想idt中填充数据后，它的内存地址还是之前那个内存地址。它的内存地址不会发生变化。
// 而它的内存地址已经存储到idt_ptr中，所以，在往idt中填充中断描述符后，使用lidt [idt_ptr]能把新idt加载到寄存器。
```

##### 创建中断门

在loader.asm中使用汇编宏创建IDT中的中断描述符，实质是门，只是把门的TYPE设置为中断描述符的TYPE而已。

创建IDT项的汇编宏有四个参数，分别是：中断例程所在的代码段的选择子、中断例程偏移量、中断门的属性（特权级+TYPE）、ParamCount。

这四个参数是中断门描述符的组成元素。在kernel中用C语言建立中断门描述符，也需要这四种元素。

1. 中断向量号。
   1. 能随意设置还是和那张中断表有关系？
2. 中断例程所在的代码段的选择子。是个常数。
3. 中断例程偏移量。
   1. 中断例程是一个汇编函数。
   2. 偏移量就是这个汇编函数的名称。
4. 中断门的TYPE。
5. 中断门的特权级。
6. ParamCount。是多少？0。

##### 异常处理函数

异常处理函数是中断函数（中断例程）的一部分。它在中断函数中接收参数（什么参数？），然后打印cs、ip、错误码。

中断函数模板的伪代码是：

```shell
Overflow:
		push 错误码
		jmp		ExceptionHandler
ExceptionHandler:
		# 调用异常处理函数
```

##### 触发中断

使用指令`ud2`，触发”指令不存在“中断。

还有一个啥中断，我不记得了。

##### 小结

目前掌握的知识，再加上不会写就看书或代码，已经足够完成本实验了。如果抛开那些字符串打印、打印整数函数，添加中断的流程并不复杂。

如果让我现在就写代码，我会按下面的流程做：

1. 825A9已经在loader中初始化了，在kernel中可以不处理。
2. 建立全局变量idt_ptr、idt[256]。
3. 在ReloadGDT中修改idt_ptr。
4. 在ReloadGDT中往idt中添加中断描述符。
5. 添加中断描述符的流程是：
   1. 中断函数。
   2. 初始化中断描述符。
      1. 中断向量号能随意设置吗？
6. 重新加载idt，lidt [idt_ptr]
7. 触发中断
   1. ud2
   2. 另外一个。我忘记了。

##### 调试

```shell
Next at t=14954679
(0) [0x00000003044b] 0008:000000000003044b (unk. ctxt): int 0x01                  ; cd01
<bochs:18> s
00014954679e[CPU0  ] interrupt(): gate not present
00014954679e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x0b)
00014954679e[CPU0  ] int_trap_gate(): selector null
00014954679i[CPU0  ] CPU is in protected mode (active)
00014954679i[CPU0  ] CS.mode = 32 bit
00014954679i[CPU0  ] SS.mode = 32 bit
```

上面的错误，是因为初始化门描述符时写错了参数：

1. 向量号和选择子弄混淆了。
2. 向量号是0，触发中断应该是`int 0x0`。
3. 选择子应该是`0x8`，而不是`0x1`。索引是1，但索引只是选择子的一部分，不等同于索引。

##### 很陌生的知识点

###### typedef void (*int_handle)()

初始化中断描述符时需要用到中断函数的偏移量。在我的代码中，中断函数的偏移量是中断函数的名称。这是如何做到的？

答案就是，初始化中断描述符的函数把偏移量声明成`int_handle`。

`int_handle`是什么数据类型？

它是这样定义的：`typedef void (*int_handle)()`。

十多天前或二十多天前，我见过这种语法，学习过，可惜忘记得干干净净。到现在，我仍没掌握这种用法。我只知道，把函数名称当偏移量需要将使用这个函数名称的形参设置为 `int_handle`类型。

###### 修改数组的元素

```c
Gate *gate = &idt[1];
gate->paramCount = 0;
```

使用上面的语法，修改gate的同时，也会修改idt[0]。

我不是很理解，`Gate *gate = &idt[0];`应该写成`Gate *gate = idt[1];`吗？

不应该，声明指针变量时，给指针变量赋值必须是内存地址。

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
        int *f = b;
        printf("f = %d\n", *f);
        return 0;
}
```

相关输出：

```shell
[root@localhost c]# gcc -o point point.c
point.c: In function 'main':
point.c:15:11: warning: initialization of 'int *' from 'int' makes pointer from integer without a cast [-Wint-conversion]
  int *f = b;
           ^
[root@localhost c]# ./point
a = 0x7ffd96ed25bc
*a = 9
b = 9
e = 9
Segmentation fault (core dumped)
```





### 最终方案

## 实践

## 调试

