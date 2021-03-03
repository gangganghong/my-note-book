# 操作系统---在内核中重新加载GDT和堆栈

## 摘要

用BIOS方式启动计算机后，BIOS先读取引导扇区，引导扇区再从外部存储设备中读取加载器，加载器读取内核。进入内核后，把加载器中建立的GDT复制到内核中。

这篇文章的最大价值也许在末尾，对C语言指针的新理解。

## 是什么

在BOOT（引导扇区）加载LOADER（加载器）。

在LOADER中初始化GDT、堆栈，把Knernel（内核）读取到内存，然后开启保护模式，最后进入Knernel并开始执行。操作系统正式开始运行了。

GDT是CPU在保护模式下内存寻址必定会使用的元素，在Kernel执行过程中也需要用到。

在内核中重新加载GDT和堆栈，是指，把存储于LOADER所使用的内存中的GDT数据和堆栈中的数据复制到Kernel所使用的内存中。关键点不是Kernel和LOADER所使用的内存，而是变量。换句话说，把存储在LOADER中的变量中的GDT数据和堆栈中的数据复制到Kernel变量中的GDT和堆栈。

LOADER是用汇编写的，“汇编中的变量”，不知道这种表述是否准确。

## 为什么

理由很简单。LOADER是用汇编语言写的，Kernel主要用C语言开发。在Kernel中使用GDT，若使用LOADER中定义的那个GDT变量（或者叫标号），光想一想就觉得很混乱。

用一句解释：C语言中使用C语言中的变量更方便。

## 怎么做

### 流程

1. 在kernel中声明变量`unsigned short gdt_ptr`，存储GDT的内存地址。
2. 使用`sgdt`指令把GDT的内地址复制到`gdt_ptr`中。
3. 在kernel中创建结构体`gdt`，存储GDT。
4. 使用内存复制函数把GDT从LOADER中设置的内存位置复制到kernel中的变量`gdt`表示的内存中。

### memcpy

它是内存复制函数。

这样实现它：

1. 原型是：`memcpy(void *dst, void *src, int size)`。
2. 核心是，把数据从`[ds:esi]`移动到`[es:edi]`。
3. 以字节为单位来复制数据，复制`size`次。
4. 用`jmp`实现循环，不用`loop`。
5. 循环终止条件是：size = 0。

```assembly
memcpy:
		push	ebp
		mov		ebp,	esp
		
		push	edi
		push	esi
		push	ecx
		push	eax
		push	ds
		push	es
		
		mov		es,		[ebp + 12] ;dst
		mov		ds,		[ebp + 8]	; src
		mov		size,	[ebp + 4]	; size
		
		mov		edi,	0
		mov		esi,	0
		mov		ecx,	size
		
.1:
		cmp		ecx,	0
		jz		.2
		mov		al,		[ds:esi]
		mov		[es:edi],		al
		inc		esi
		inc		edi
		dec		ecx
.2:	
		pop		es
		pop		ds
		pop		eax
		pop		ecx
		pop		esi
		pop		edi
		pop		ebp
		ret
```

### gdt

```c
typedef struct {
  	unsigned  short  limitLow;
  	unsigned  short  baseAddressLow;
  	unsigned	char	 baseAddressMid;
  	unsigned	char	 attribute1;
  	unsigned	char	 attribute_limit;
  	unsigned	char	 baseAddressHigh;
}Descriptor;

Descriptor gdt[128];
```

### 堆栈

```assembly
[SECTION .bss]
StackSpace		resb			2 * 1024
StackTop:

mov		esp,	StackTop
```

不理解。

### 代码

C语言

```c
// 声明一个char数组，存储GDT的内存地址
unsigned	char	gdt_ptr[6];
```

nasm汇编

```assembly
; 使用C语言中声明的变量gdt_ptr
extern gdt_ptr
; 把寄存器gdtr中的数据复制到变量gdt_ptr中
sgdt	[gdt_ptr]
```

然后在C语言中把LOADER中的GDT复制到C语言中的gdt变量中。

```c
memcpy(&gdt, 
       (void *)((*)(int *)(&gdt_ptr[2])), 
       (*)((int *)(&gdt_ptr[0]))
      );
short *gdt_limit = &gdt_ptr[0];
int	*gdt_base = &gdt_ptr[2];

*gdt_limit = 128 * sizeof(Descriptor) - 1;
*gdt_base = (int) &gdt;
```

#### 难点解读

#### memcpy的参数

上面的那段代码，理解起来难度不小。

```assembly
memcpy(&gdt, 
       (void *)((*)(int *)(&gdt_ptr[2])), 
       (*)((short *)(&gdt_ptr[0]))+1
      );
```

`memcpy`的第一个参数是目标内存地址，是一个指针类型变量，赋值应该是一个内存地址，所以用`&`取得变量gdt的内存地址。

1. 理解`(void *)((*)(int *)(&gdt_ptr[2]))`：
   1. 第二个参数是源数据的内存地址，是GDT的物理地址。
   2. 它存储在`gdt_ptr`的后6个字节中。
   3. `&gdt_ptr[2]`获取gdt_ptr的第3个元素`gdt_ptr[2]`的物理地址。
   4. 前面的`(int *)`将这段物理地址强制类型转换为一个指针，这个指针的数据类型是`int *`。
   5. 数据类型是`int *`有三层含义：
      1. 这个数据是一个指针。
      2. 这个数据的值是一个内存地址。
      3. 这个内存地址是一个4字节（int类型占用4字节）内存区域的初始地址。
   6. `&gdt_ptr[2]`是一个内存地址，用`(int *)`将它包装成或强制转换成指针类型。
   7. 再用`*`运算符，是获取这个内存地址指向的内存区域中的数据。
   8. 这个数据是`int`类型，占用4个字节。这4个字节的初始地址是`&gdt_ptr[2]`。这是最关键的一句。
   9. 为什么最后还要用`void *`？
      1. 因为，这4个字节中存储的那个`int`数据又是一个内存地址，因此，需要再次包装成一个指针。
      2. 因为，`memcpy`对参数的数据类型要求是`void *`。
      3. 究竟是哪个原因，我也不知道。
2. 理解：`(*)((short *)(&gdt_ptr[0]))+1`
   1. 为什么要加1？`gdt_ptr`的低2位保存的是GDT的字节偏移量的最大值，是GDT的长度减1。
   2. `&gdt_ptr[0])`是`gdt_ptr[0])`的内存地址AD。
   3. `(short *)&gdt_ptr[0])`用AD创建一个指针变量。
      1. 这个指针变量指向一块内存。
      2. 这块内存占用2个字节。
      3. 这块内存的初始地址是`&gdt_ptr[0])`，即`gdt_ptr[0])`的内存地址。
      4. `(short *)&gdt_ptr[0])`实质是指代`&gdt_ptr[0]、&gdt_ptr[1]`这两小块内存。
   4. `(*)((short *)(&gdt_ptr[0]))`是`&gdt_ptr[0]、&gdt_ptr[1]`这两小块内存中的值，即`gdt_ptr[0]、gdt_ptr[1]`。
   5. 为什么不需要像第二个参数一样在前面再加上一个`(void *)`？
      1. 因为，第4步的结果是一个short类型的整型数（short能称之为整型吗？），不是内存地址，不需要强制类型转换。

#### 其他

```c
short *gdt_limit = &gdt_ptr[0];
int	*gdt_base = &gdt_ptr[2];

*gdt_limit = 128 * sizeof(Descriptor) - 1;
*gdt_base = (int) &gdt;
```

```c
short *gdt_limit = &gdt_ptr[0];
int	*gdt_base = &gdt_ptr[2];
```

这段代码创建了两个变量并赋值，获取了GDT的界限和地址。可是紧接着又有下面两句，是对GDT的界限重新赋值。

```c
*gdt_limit = 128 * sizeof(Descriptor) - 1;
*gdt_base = (int) &gdt;
```

这两段代码的功能重复了吗？

让我们先看另外一段代码。

```c
#include <stdio.h>

int main(int argc, char **argv)
{
        int b = 8;
        printf("b = %d\n", b);
        int *a = &b;
        *a = 9;
        printf("b = %d\n", b);

        return 0;
}
```

执行结果是：

```shell
MacBook-Pro:my-note-book cg$ ./test
b = 8
b = 9
```

第8行`*a = 9;`修改`*a`的值，同时也修改了`b`的值，因为第7行`int *a = &b;`。

再回头看

```c
short *gdt_limit = &gdt_ptr[0];
int	*gdt_base = &gdt_ptr[2];

*gdt_limit = 128 * sizeof(Descriptor) - 1;
*gdt_base = (int) &gdt;
```

`*gdt_base`指向`gdt_ptr[2]`为初始地址的4个字节的连续的内存空间AD，修改`*gdt_base`，实质是修改AD中的数据。`int	*gdt_base = &gdt_ptr[2];`的作用是让`*gdt_base`指向AD；`*gdt_base = (int) &gdt;`是修改AD中的数据，从业务逻辑的角度看，是把gdt的内存地址写入AD中。为什么要这样做？回忆一下我们的目的是什么？把存储了GDT的C语言中变量的内存地址存储到gdt_ptr中。

## 意外收获

在理解上面那个比较复杂的指针参数的过程中，我对指针有了新的理解。

`int a;`，要求CPU（不知道执行者是CPU还是操作系统）为`a`分配四个字节的内存空间，存储数据。

`int *a;`，要求CPU为`a`分配四个字节（第一片四字节内存空间，记作A），在这四个字节中存储一个内存地址，这个内存地址指向另外一个四字节的内存区域（记作B）。`int *a`的含义是，指向B中的`int`类型数据。

```assembly
char days[5];
（short *)(&days[2]);
```

`（short *)(&days[2]);`的含义是：

1. 指向一片内存区域addr，这片内存区域的长度是连续的2个字节（short是2个字节）。
2. addr的初始地址是days[2]的内存地址，所以，这片内存是`&days[2]，&days[3]`。

