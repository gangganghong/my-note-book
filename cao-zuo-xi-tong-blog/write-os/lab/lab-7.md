# 进入内核后切换堆栈和GDT

## 理论

### 回忆

在loader中进入内核。进入内核前，堆栈、GDT、IDT、LDT都建立好了。开发内核，主要用C语言。切换堆栈和GDT，就是把loader中建立好的堆栈、GDT中的数据复制到用C语言写的堆栈和GDT中。

#### 为什么要切换

为什么要这样做？

在C语言写的内核中也需要用到GDT、堆栈等。在C语言中直接使用汇编写的GDT、堆栈等，我不知道应该用什么方法，即使有方法，应该也是非常麻烦的方法。而在C语言中使用C语言写的GDT、堆栈等，非常方便。

第一个问题，需要切换的除了堆栈和GDT，还有什么？IDT、LDT需要切换吗？

第二个问题，在C语言中，怎么表示堆栈？

#### 怎么切换

“切换”，是把loader中建立的GDT、堆栈的数据复制到内核中的GDT、堆栈变量中。内核中的GDT、堆栈变量是用C语言写的。

用C语言表示GDT，不是难点，但我仍然没有记住GDT描述符的结构。对着书上的GDT描述符结构图，我能比较快写好描述符。

用C语言表示堆栈，怎么表示？

GDT是一块存储若干个全局描述符的内存区域。找到GDT的初始地址和GDT的长度，就能把这块内存的数据复制到新的GDT变量中。

GDT的初始地址和长度，存储在专用寄存器GdtPtr中。

有一个疑问：GDT的初始地址和长度存储在专用寄存器GdtPtr中，同时，也存储在汇编标量GdtPtr中。我应该从汇编标量还是专用寄存器中获取数据呢？

在C语言中获取寄存器中的数据，我不知道什么方法能达到目的，只能从汇编标量中获取数据了。

汇编标量能不能直接在C语言中当变量使用？我记得于上神的代码是这样做的。

首先，要知道怎样才能在C语言中获取在汇编中建立的GDT数据。

其次，把从汇编中获取到的GDT数据复制到C语言GDT变量中。这段代码的难度让我到现在还心有余悸。难点是，C语言中的嵌套指针使用。

*当初花了那么多时间理解切换GDT的代码，现在又忘记了，真恼火！*

随便想想吧，怎么把汇编中的GDT复制到C语言中的GDT？

创建一个struct（记作Des）存储GDT GDT中的描述符，GDT（记作GDT）是这个struct的数组。

GDT有多少个元素？在C语言中，创建数组需要确定元素个数（是这样吗）。假设GDT中N个元素，严谨一些，在C语言中，GDT存储在GDT[N]数组中。

切换GDT，就是把汇编GDT中的数据复制到GDT[N]中。

不必把GDT[N]当数组，把GDT[N]当一段内存空间P，知道这段内存空间的初始地址。

已知汇编GDT的初始内存地址和长度，把这段内存中的数据复制到P中。

复制数据使用`Memcpy`。`Memcpy`是使用C语言的内置函数还是使用我写的`Memcpy`。

*又很沮丧。虽然我写过Memcpy，可现在我并没有一看见Memcpy就知道怎么实现Mempcy。*

我只能想到这么多了。

#### 回忆1

*看了以前的笔记和书*

在C语言中创建一个变量`gdt_ptr`，然后导入汇编代码中。用什么指令？是`extern gdt_ptr`吗？

在汇编代码中，用`sgdt [gdt_ptr]`加载到变量`gdt_ptr`中。

`gdt_ptr`从汇编代码中获得指向GDT的“元数据”（有更好的专业术语，忘记了，先这样描述吧），然后在C代码中使用。

把汇编代码中创建的GDT用`Memcpy`复制到C代码中的GDT中。问题的实质是下面两件事情：

1. 实现Memcpy。
2. 从gdt_ptr中获取Memcpy的两个参数。

我之前的确实现过Memcpy，但现在的确忘记了。让我马上实现Memcpy，既不能一次性说出思路，也不能流畅地写出代码。我决定直接复用原来的代码。有时间，再单独实现Memcpy。不可为了已经实现过的函数而拖慢进度。

##### 实现Memcpy的思路

简单回顾Mempcy的实现思路：

1. 原型：`Memcpy(void *dst, void *src, int size)`。三个参数分别是：目的地址，源地址，被复制的数据的长度。
2. nasm函数模板
   1. 在开头把函数中会改变的寄存器中的值存入堆栈。
   2. 从栈中获取函数的参数。
      1. 函数的参数的入栈顺序是从右往左。例如，函数有三个参数，第三个参数最先入栈，然后第二个参数入栈，然后第一个参数入栈。
      2. bp是堆栈的栈顶，第三个参数是[bp+4]，第二个参数是[bp+8]，第三个参数是[bp+12]。
      3. [bp+0]是`call Memcpy`的下一条指令。
   3. 在开头的入栈的值，按和入栈顺序相反的顺序出栈。
   4. 使用`ret`。一定不能漏掉。它把下一条指令的地址等更新到ip等寄存器中。
3. 逐字节把数据从[es:edi]复制到[ds:esi]中。如果不行，颠倒二者的顺序。
   1. 使用`lodsb`把数据从[es:edi]加载到ax中，然后从ax中把数据复制到[ds:esi]中。
   2. 手工更新 ecx ，再配合 jmp 来实现循环。

##### 最大难点

是从gdt_ptr中获取GDT的基地址和限制，分别记作base和limit。

limit是GDT的长度减去1。

这个问题，实质是：如何把一个数组拆分成两个部分使用。

存储GDT“元数据”的变量是一个数组，`char gdt_ptr[6]`，48个位，与GDT“元数据”所需内存的长度相同。

这个数组的前2个元素存储GDT的limit，后4个元素存储GDT的基地址。

这是个很有用的用法。

获取数组的前2个元素中的数据，`(short *)(&gdt_ptr[0])`。

获取数组的后4个元素中的数据，`(int *)(&gdt_ptr[2])`。

怎么理解`(short *)(&gdt_ptr[0])`？只解释一个，二个表达式的理解方式相同。

1. `*`表示这个变量是指针。
   1. 一个变量是指针，意味着，在内存中给这个变量分配了4个字节。
   2. 这4个字节存储一个内存地址。
   3. 这个内存地址指向一片内存。“一片内存”的意思是，一段内存的初始地址已知，这个初始地址的后面的内存是不是属于这个变量，还需要额外的条件来判断。
   4. 这个额外条件就是`*`前面的数据类型，例如本例中的`short`。
   5. `short *`的意思是，从初始地址开始往后再数2个字节（包括初始地址指向的这个字节在内共2个字节），这2个字节的内存中存储的数据就是`(short *)(&gdt_ptr[0])`。
2. `&gdt_ptr[0]`，怎么理解？
   1. 等价于`gdt_ptr`。
3. 内存地址必须包装成指针类型变量。

##### 使用Memcpy

`Memcpy(&gdt, (void *)((int *)(&gdt_ptr[2])), (short *)(&gdt_ptr[0]));`

第2个参数前面加上`(void *)`是为了和函数的形参保持一致。

第3个参数为啥不需要加上`int`？也许是因为第3个参数的值本来就是`int`类型。

上面的用法是语句是错误的。正确的应该是：

```c
Memcpy(&gdt,
      (void *)(*((int *)(&gdt_ptr[2]))),
       *((short *)(&gdt_ptr[0])) + 1
      );
```

主要差异是，又加了一层星号`*`。

`(int *)(&gdt_ptr[2])`，是一个指向int数据的指针。在它上面再加上一层星号，就成了`*(int *ptr)`。这个语句是取出指针指向的数据。

在Memcpy中，如果不加上外层星号，那么实参是一个指针，而形参要求指针是一个数据（而并非指针）。这种解释，需要再验证。

```c
// warning: initialization of 'int *' from 'int' makes pointer from integer without a cast [-Wint-conversion]
int *d = 8;
```

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
        int *d = &c;
        printf("d = %d\n", d);
        printf("d = %d\n", *d);

        return 0;
}
```

输出是：

```shell
[root@localhost c]# ./point
a = 0x7fff97d06b7c
*a = 9
b = 9
d = -1747948680
d = 9
```



##### 更新gdt_ptr

gdt_ptr存储用汇编创建的GDT的“元数据”。把汇编代码中的GDT复制到C代码后，gdt_ptr中的数据应该更新成C代码中GDT的“元数据”。

C代码创建GDT：`全局描述符 gdt[描述符数量]`。

`int *gdt_base = (int *)(&gdt_ptr[2])`

`short *gdt_limit = (short *)(&gdt_ptr[0])`

使用`(int *)、(short *)`的目的是，使用四个字节、二个字节，不是因为前面创建的是指针变量。创建指针变量，只需要赋值时赋予内存地址。

不知道上面的写法是否正确。

`*gdt_base = &gdt`

`*gdt_limit = 描述符数量 * sizeof(描述符) - 1`

这两个语句，也是难点。

于上神的代码是（我写成了伪代码）：

```c
*gdt_base = (int)&gdt;		// (int)可选，内存地址必定是32位。这个说法，需要验证。
*gdt_limit = 描述符数量 * sizeof(描述符) - 1;// 这是C语言中的用法，gdt_limit = &var，*gdt_limit = var
```



### 最终方案



## 实践

代码在：/home/cg/os/pegasus-os/v15。

流程如下：

1. 让整个系统进入内核。可能需要清理掉其他实验代码。
2. 在loader中加载GdtPtr。就在跳入内核的语句前面吧。
3. 在kernel_main.c中：
   1. 创建函数ReloadGDT，重新加载
   2. 创建变量 char gdt_ptr[6]；在loader.asm中导入整个变量。
4. 把Mempcy复制到string.asm中。
5. 修改Makefile
   1. 不知道应该怎么修改。



找到一个能加载内核并执行的版本，花了很长时间。原因是，我不记得每个版本的代码情况。需详细记录啊。时间久了，更加不知道它们的情况。花了22分。打开文件、检查。

kernel_main.c 怎么写？

1. 函数名是main吗？不是。
2. 如何编译？把string.s、kernel_main.c编译成elf文件string.o、kernel_main.o；然后把它们一起编译成kernel.bin。

Kernel.asm中

代码大概是这样的：

```assembly
sgdt	[gdt_ptr]
call	ReloadGDT
lgdt	[gdt_ptr]

ret
```

怎么导入gdt_ptr和Memcpy？



string.asm专门存放字符串相关的自定义函数，这个文件怎么写？

1. 直接写函数吗？
2. 弄一些section?
3. 参考于上神的吧。

模板是：

```assembly
[SECTION .txt]

global Mempcy

Mempcy:

			; some code
			ret
```



## 调试

代码写完了，但是，

`ld -s -Ttext 0x30400  -o kernel.bin  kernel.o string.o kernel_main.o -m elf_i386` 导致显示器被清屏。

`ld -s -Ttext 0x30400  -o kernel.bin  kernel.o -m elf_i386`，能正常显示。



## 参考

1. [^/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/dos/switch-gdt-stack.md]: 操作系统---在内核中重新加载GDT和堆栈

   