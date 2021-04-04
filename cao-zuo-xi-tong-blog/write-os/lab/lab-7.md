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

### Mempcy导致被显示器被清屏

代码写完了，但是，

`ld -s -Ttext 0x30400  -o kernel.bin  kernel.o string.o kernel_main.o -m elf_i386` 导致显示器被清屏。

`ld -s -Ttext 0x30400  -o kernel.bin  kernel.o -m elf_i386`，能正常显示。

使用`ld -s -Ttext 0x30400  -o kernel.bin  kernel.o string.o kernel_main.o -m elf_i386`编译，CPU执行到了内核中吗？

在进入内核的跳转语句前加上断点。

是 call InitKernel 导致显示器被清屏。

InitKernel 中某次调用 Memcpy 导致显示器被清屏。

能确定，loader.asm中的Memcpy有问题。出现问题的两个条件：

1. Memcpy
2. Kernel_main.c中声明全局变量 char gdt_ptr[6]。

使用下面的方法进一步定位出错的地方：

1. 在Memcpy中设置断点，断点位置分别是结尾和中间。
2. 去掉loader.asm中的其他断点，方便查看Memcpy中的断点。

基本解决了问题，错误原因是：在ecx等于0时仍然将ecx递减，导致陷入死循环。

*大概花了1个小时26分解决这个问题。我比较满意。这是一次没有严重浪费时间、思路清晰的解决问题的典范过程。*

屏幕被清屏，我是这样处理的：

1. 无明确思路地小修小改，然后运行，没找出任何线索。唯一让我觉得代码没能正常运行的症状是：屏幕被清屏。

2. 逐个减少参与kernel.bin编译的文件：string.o、kernel_main.o。

   1. 发现，没有kernel_main.o参与编译，就能正常执行。
   2. 进一步发现，在kernel_main.c中创建全局变量就能重现问题。
      1. 修改char gdt_ptr[6]为char gdt_ptr2[6]。仍能重现问题。
      2. 去掉这个变量或去掉全局变量，在函数内创建同名局部变量，不能重现问题。
      3. 这个线索很容易误导我，让我以为问题出在kernel_main.c中。
      4. 我对比了于上神的代码，确信，在kernel_main.c中能写任意合法代码。

3. 我决定，断点调试。

   1. 使用”二分法“思想，逐步确定，问题出在InitKernel--->Memcpy中。

   2. 是由于loader.asm 和 kernel.bin 中存在同名的Memcpy函数吗？

   3. 修改了string.asm中的Memcpy，仍能重现问题，因此，错误原因不是Memcpy重名。

   4. 去掉所有断点，只在Memcpy的中间和和结尾部分设置断点，记录在第N次断点后出现问题。

   5. 重新运行，达到第N次断点时，单步调试，发现，

      1. ecx等于0时仍被减去1，变成`0xffffff`（总之是溢出了，可能不是这么多f）。

      2. 在ecx减1后，有一个判断，根据ecx是否等于1决定是否跳转。

      3. 由于输入给Memcpy的数据特殊，ecx变成了`0xfffff`后和1比较，导致死循环。看下面的节选代码。

         ```assembly
         .1:
                 mov byte al, [ds:esi]
                 mov [es:edi], al
         
                 inc esi
                 inc edi
                 cmp ecx, 0
                 jz .2
                 dec ecx
         
                 cmp ecx, 0
                 jz .2
                 jmp .1
         ```

         死循环反复执行`.1`这段代码。总是执行这段代码为何会清屏，我不知道。修改ecx溢出的漏洞后，问题就消失了。

*的确很满意解决这个问题的过程。第一，没有浪费时间。第二，思路清晰，没有漫无目的地胡乱调试。*

解决这个问题，不需要专业知识，只需要：

1. bochs的断点调试、单步执行、查看寄存器。xchg、s、r。
2. 二分法。
3. sed快速清空文件中的断点。

### 重载GDT成功了吗？

验证方法：

1. 通过断点调试获取一些数据：
   1. GdtPtr
   2. 第一个描述符
   3. 第二个描述符
2. 与上面相同。

具体做法：

1. 在loader.asm的`lgdt [GdtPtr]`前面设置断点，获取GdtPtr。
2. 进入保护模式后，使用sreg获取GDT base，然后使用`xp`查看前二个描述符。
3. 在kernel.asm中的`lgdt [GdtPtr]`前面设置断点，方法与第1、2步相同。

```shell
(0) [0x00000002029c] 2000:029c (unk. ctxt): lgdt ds:0x017f            ; 0f01167f01
<bochs:3> sreg
es:0x9000, dh=0x00009309, dl=0x0000ffff, valid=1
	Data segment, base=0x00090000, limit=0x0000ffff, Read/Write, Accessed
cs:0x2000, dh=0x00009302, dl=0x0000ffff, valid=1
	Data segment, base=0x00020000, limit=0x0000ffff, Read/Write, Accessed
ss:0x0000, dh=0x00009300, dl=0x0000ffff, valid=7
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
ds:0x2000, dh=0x00009302, dl=0x0000ffff, valid=7
	Data segment, base=0x00020000, limit=0x0000ffff, Read/Write, Accessed
fs:0x0000, dh=0x00009300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
gs:0xb800, dh=0x0000930b, dl=0x8000ffff, valid=7
	Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0000, dh=0x00008200, dl=0x0000ffff, valid=1
tr:0x0000, dh=0x00008b00, dl=0x0000ffff, valid=1
gdtr:base=0x00000000000f9af7, limit=0x30
idtr:base=0x0000000000000000, limit=0x3ff
<bochs:4> xp /1gx 0x2000:0x017f
[bochs]:
0x000000000002017f <bogus+       0>:	0xc88c0002013f003f

<bochs:6> sreg
es:0x9000, dh=0x00009309, dl=0x0000ffff, valid=1
	Data segment, base=0x00090000, limit=0x0000ffff, Read/Write, Accessed
cs:0x0008, dh=0x00cf9b00, dl=0x0000ffff, valid=1
	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
ss:0x0000, dh=0x00009300, dl=0x0000ffff, valid=7
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
ds:0x2000, dh=0x00009302, dl=0x0000ffff, valid=7
	Data segment, base=0x00020000, limit=0x0000ffff, Read/Write, Accessed
fs:0x0000, dh=0x00009300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
gs:0xb800, dh=0x0000930b, dl=0x8000ffff, valid=7
	Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0000, dh=0x00008200, dl=0x0000ffff, valid=1
tr:0x0000, dh=0x00008b00, dl=0x0000ffff, valid=1
gdtr:base=0x000000000002013f, limit=0x3f
idtr:base=0x0000000000000000, limit=0x3ff

<bochs:7> xp /1gx 0x000000000002013f
[bochs]:
0x000000000002013f <bogus+       0>:	0x00000000
<bochs:8> xp /1gx 0x20147
[bochs]:
0x0000000000020147 <bogus+       0>:	0xcf9b000000ffff
<bochs:9> xp /1gx 0x2014f
[bochs]:
0x000000000002014f <bogus+       0>:	0xf98020460ffff


```

在kernel.bin中

```shell
(0) [0x00000003041d] 0008:000000000003041d (unk. ctxt): lgdt ds:0x00032500        ; 0f011500250300
<bochs:8> sreg
es:0x0030, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
cs:0x0008, dh=0x00cf9b00, dl=0x0000ffff, valid=1
	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
ss:0x0030, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
ds:0x0030, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
fs:0x0030, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
gs:0x003b, dh=0x0000f30b, dl=0x8000ffff, valid=7
	Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0000, dh=0x00008200, dl=0x0000ffff, valid=1
tr:0x0000, dh=0x00008b00, dl=0x0000ffff, valid=1
gdtr:base=0x000000000002013f, limit=0x3f
idtr:base=0x0000000000000000, limit=0x3ff
<bochs:9> xp /1gx 0x30:0x32500
[bochs]:
0x0000000000032500 <bogus+       0>:	0x53fd0003200004ff


<bochs:12> sreg
es:0x0030, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
cs:0x0008, dh=0x00cf9b00, dl=0x0000ffff, valid=1
	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
ss:0x0030, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
ds:0x0030, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
fs:0x0030, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
gs:0x003b, dh=0x0000f30b, dl=0x8000ffff, valid=7
	Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0000, dh=0x00008200, dl=0x0000ffff, valid=1
tr:0x0000, dh=0x00008b00, dl=0x0000ffff, valid=1
gdtr:base=0x0000000000032000, limit=0x4ff
idtr:base=0x0000000000000000, limit=0x3ff

<bochs:13> xp /1gx 0x32000
[bochs]:
0x0000000000032000 <bogus+       0>:	0x00000000
<bochs:14> xp /1gx 0x32008
[bochs]:
0x0000000000032008 <bogus+       0>:	0xcf9b000000ffff
<bochs:15> xp /1gx 0x32010
[bochs]:
0x0000000000032010 <bogus+       0>:	0xf98020460ffff

```

描述符是正确的，但是，`gdtr:base=0x0000000000032000, limit=0x4ff`中的`limit`不相同。~~二者应该相同才对。~~

二者本来就不相同。

在loader中，GDT中包含8个描述符，limit = 8 * 8 - 1 = 3f。

在loader中，GDT的limit = 128 * 8 - 1 = 3ffh。

注意，limit的计算单位是字节而不是bit，不要弄成了bit。我一开始认为limit的计算单位是bit。

正确的打印数据：

loader

```shell
<bochs:3> sreg
es:0x9000, dh=0x00009309, dl=0x0000ffff, valid=1
	Data segment, base=0x00090000, limit=0x0000ffff, Read/Write, Accessed
cs:0x0008, dh=0x00cf9b00, dl=0x0000ffff, valid=1
	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
ss:0x0000, dh=0x00009300, dl=0x0000ffff, valid=7
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
ds:0x2000, dh=0x00009302, dl=0x0000ffff, valid=7
	Data segment, base=0x00020000, limit=0x0000ffff, Read/Write, Accessed
fs:0x0000, dh=0x00009300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
gs:0xb800, dh=0x0000930b, dl=0x8000ffff, valid=7
	Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0000, dh=0x00008200, dl=0x0000ffff, valid=1
tr:0x0000, dh=0x00008b00, dl=0x0000ffff, valid=1
gdtr:base=0x000000000002013f, limit=0x3f
idtr:base=0x0000000000000000, limit=0x3ff
```

kernel

```shell
<bochs:17> sreg
es:0x0030, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
cs:0x0008, dh=0x00cf9b00, dl=0x0000ffff, valid=1
	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
ss:0x0030, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
ds:0x0030, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
fs:0x0030, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
gs:0x003b, dh=0x0000f30b, dl=0x8000ffff, valid=7
	Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0000, dh=0x00008200, dl=0x0000ffff, valid=1
tr:0x0000, dh=0x00008b00, dl=0x0000ffff, valid=1
gdtr:base=0x0000000000032000, limit=0x3ff
idtr:base=0x0000000000000000, limit=0x3ff
```

计算出来的GDT的limit和打印出来的limit一致。这说明，在内核中重新加载GDT成功了。

### C语言的sizeof

```c
char gdt_ptr2[6];
int main(void)
{
        int arr[2] = {1,2};
        printf("arr[0] = %d\n", arr[0]);

        typedef struct{
        unsigned short seg_limit_below;
        unsigned short seg_base_below;
        unsigned char  seg_base_middle;
        unsigned char seg_attr1;
        unsigned char seg_limit_high_and_attr2;
        unsigned char seg_base_high;
}Descriptor;

        printf("Descriptor size = %d\n", sizeof(Descriptor));

        typedef struct
{
    int a;
    char b;
    double c;
} Simple2;


        printf("int size = %d\n", sizeof(int));
        printf("char size = %d\n", sizeof(char));
        printf("double size = %d\n", sizeof(double));
        printf("Simple2 size = %d\n", sizeof(Simple2));

        struct {char b; double x;} a;
        printf("a size = %d\n", sizeof(a));
 				
  			return 0;
}
```



```shell
[root@localhost v15]# ./t
arr[0] = 1
Descriptor size = 8
int size = 4
char size = 1
double size = 8
Simple2 size = 16
a size = 16
```

先贴一张图。

![image-20210404145252149](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210404145252149.png)

sizeof应用于不同的数据类型计算结果的方式不同，在  http://c.biancheng.net/cpp/html/932.html 有很详细的理解。关于struct的举例是错误的。

那篇文章写得很全面。不过，看一次甚至看两三次，也难记住。由于不需要考试，我只记住下面几点就够用了：

1. sizeof应用于上图中的数据类型，结果是这个数据类型所占用的字节数。
2. sizeof应用于联合体（Union），结果是联合体中所占空间最大的成员的长度（单位也是字节数）。
3. sieof应用于struct，比较复杂，
   1. 结果并不是成员的长度之和。
   2. 受对齐规则影响。
   3. 为一个成员M分配内存时，用M的偏移量O除以sizeof(M)。
      1. 没有余数时，紧挨着前一个成员分配内存。
      2. 有余数时，要在前一个成员和M之间再分配(O/sizeof(M) + 1)*sizeof(M) - O个字节。
      3. 这是为了对齐分配的空间，没有意义。

## GDT重载小结

### 时间消耗点

重载GDT有三大时间消耗点：

1. Mempcy的三个参数
   1. C语言的嵌套指针
   2. 如何使用一个数组的局部，比如前2个元素、后4个元素。
2. 验证GDT切换是否成功
   1. 想出思路
   2. 计算GDT的limit
      1. C语言sizeof(struct)的结果，受”对齐“影响。
      2. limit的单位是字节，不是bit。
3. Memcpy中ecx溢出导致屏幕被清屏。

### 调试GDT的limit的过程

1. 重载GDT前后的limit的值不同。
2. 认为二者应该相同，后来，确定，二者应该不同。
   1. 在loader中只创建了8个全局描述符。
   2. 在kernel中GDT中包含128个全局描述符。
3. 获取到的重载后的GDT的limit是十六进制数，把它转成十进制数，加1除以64。
4. 结果并不是128。
5. 后来我确定，limit + 1 后除以描述符的长度，结果必须是128。
6. (limit + 1) / 128 = 8。根据实际结果，计算出结构体Descriptor的长度是8。
7. 我认为不应该是这个值，而应该是64。
8. 在独立的C程序中计算sizeof(Descriptor)，结果仍是8。
9. 我的注意力转移到C语言中的sizeof。
10. 弄清sizeof作用于结构体的计算方法后，所有数据都能对应得上。

## 重载堆栈

在loader中很少使用堆栈，不创建堆栈也能够使用堆栈。

完全不觉得在内核中需要重载堆栈。

重载堆栈是啥意思？把loader中的堆栈中的数据复制到内核的堆栈中吗？切换堆栈后，有什么用吗？

上面的问题，我完全不清楚，只能看书。看，然后写；重复几次，就当作我已经掌握这块知识了。

切换堆栈很容易，只有几个陌生的知识点。

### 陌生知识

#### hlt

在汇编语言中，本指令是处理器**“**暂停**”**指令。

使程序停止运行，处理器进入暂停状态，不执行任何操作，不影响标志。当复位（外语：RESET）线上有复位信号、CPU响应非屏蔽中断、CPU响应可屏蔽中断3种情况之一时，CPU脱离暂停状态，执行HLT的下一条指令。

*仍然不知道是啥意思，有啥必要使用这条指令。*

执行效果没啥不同，只是多了一句：`00014952675i[CPU0  ] WARNING: HLT instruction with IF=0!`。

#### popfd

把堆栈栈顶元素更新到eflags寄存器。eflags寄存器存储32位数据，栈顶中的元素正好是32位数。

pushfd把eflags寄存器中的值存储到堆栈中。和其他寄存器一样，在某段代码中修改eflags前先把eflags中的值存储起来，等待修改eflags的那段代码执行完后，再把存储在堆栈中的值更新到eflags中。

还有其他操作堆栈的指令，内容有点多，不必全部记住。

PUSHAD 指令按照 EAX、ECX、EDX、EBX、ESP（执行 PUSHAD 之前的值）、EBP、ESI 和 EDI 的顺序，将所有 32 位通用寄存器压入堆栈。

POPAD 指令按照相反顺序将同样的寄存器弹出堆栈。与之相似，PUSHA 指令按序（AX、CX、DX、BX、SP、BP、SI 和 DI）将 16 位通用寄存器压入堆栈。

POPA 指令按照相反顺序将同样的寄存器弹出堆栈。在 16 位模式下，只能使用 PUSHA 和 POPA 指令。

### 创建堆栈

```assembly
[SECTION .bss]
Stack				resb			1024*2
StackTop:
```

`[SECTION .bss]`改成其他名字也行，使用它是习惯。

唯一的一点，栈顶。

我记得，之前创建堆栈，栈顶 = 栈的长度 - 1。

resb是不是把堆栈初始化成了0？不是。已经验证过了。

于上神的代码中，有这样一段：

```assembly
jmp	SELECTOR_KERNEL_CS:csinit
csinit:		; “这个跳转指令强制使用刚刚初始化的结构”——<<OS:D&I 2nd>> P90.

	push	0
	popfd	; Pop top of stack into EFLAGS

	hlt
```

`jmp	SELECTOR_KERNEL_CS:csinit`，为何如此？

## 参考

1. [^/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/dos/switch-gdt-stack.md]: 操作系统---在内核中重新加载GDT和堆栈

   