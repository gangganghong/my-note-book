# 内核雏形

## 在Linux下用汇编写Hello World

编译链接方法：

```shell
nasm -f elf hello.asm -o hello.o
# -m elf_i386 ，在64位系统上编译出32位程序需要使用
ld -s hello.o -o hello -m elf_i386
```

代码：

```assembly
[section .data]
strHello	db	"Hello, OS World!"
strHelloLenght	equ $ - strHello

[section .text]
global _start

_start:
	mov eax, 4
	mov ebx, 1
	mov ecx, strHello
	mov edx, strHelloLength
	int 0x80
	mov eax, 1
	mov ebx, 0
	int 0x80
```

在linux下使用汇编编程，使用系统调用的套路，掌握两个知识点：

1. 使用中断`int 0x80`，并给这个中断提供参数。它的参数通过寄存器传递，除系统调用调用的函数（例如，sys_write）之外，还需指定被调用的函数本身。
2. 被调用的函数本身的代号，放在eax中，其他参数依次放到这些寄存器中：ebx（第一个参数）、ecx（第二个参数）、edx（第三个参数）、esi（第四个参数）、edi（第五个参数）。

> 看看吧。只要投入时间学起来，终究是比不学习要有点收获。
>
> 这个知识点，一点都不难。可它是学习其他知识的基础。不先学这个知识点，当我遇到建立在这个知识点基础上的知识时，就会感受到它的难了。就像写一个英语句子好像不难，但不会写英语单词的时候，写一个英语句子会很难。

这段程序有两个节（section），一个用来存放数据，一个用来存放代码。

程序的入口是`_start`，这是固定的。必须使用`global`导出`_start`，才能在使用`ld`链接时被找到。

## Linux下混合使用C和汇编

汇编。foo.asm

```assembly
extern choose

[section .data]
num1	dd	3
num2	dd	5

[section .text]
global	_start
global	my_print

_start:
	push	dword	num1
	push	dword num2
	call	choose
	add	esp, 8
	
	mov eax,	1				; sys_exit
	mov ebx, 0
	int 0x80
	

my_print:
	mov eax, 	4				; sys_write
	mov ebx,	1
	mov ecx,	[esp + 4]	; msg
	mov edx,	[esp + 8]		; len
	int 0x80
	ret
	
```



C代码。bar.c

```c
void my_print(char *msg, int len);

int choose(int a, int b)
{
		if(a >= b){
      my_printf("The 1thd one\n", 13);
    }else{
      my_printf("The 2thd one\n", 13)
    }
  
  	return 0;
}
```



编译

```shell
nasm -o foo.o foo.asm
gcc -c -o bar.o bar.c -m32
ld -s foo.o bar.o foo -m elf_i386
```



达到了这个标准，这块知识，就算掌握了，也够用了。唯一欠缺的是，把编译命令写到Makefile中。

## ELF

elf文件由elf头、程序头表、节、节头表组成。只有elf头的位置是固定的，其他元素的位置不固定，在elf头中设置。

## 用loader加载ELF

像boot加载loader一样，把elf读入内容就可以了吗？

回答，是也不是。首先，把内核读入内存，然后再处理。怎么处理呢？

根据ELF文件的program header table中的值把段放入相应的位置。

具体怎么做？

### 重新放置内核

使用这个函数，`memcpy(p_vaddr, BaseOfLoader + p_offset, p_size)`。但是，需要自己实现这个函数。

`p_vaddr`，是段在内存中的虚拟地址。`p_offset`，是段在文件中的偏移量。`p_size`，是段在文件中的长度。

> 为何是虚拟地址？虚拟地址是啥？
>
> 虚拟地址，就是说，它不是内存的物理地址，需要经过一些转换才能变成物理地址。

由`ld`生成的可执行文件的中`p_vaddr`的值类似于`0x8048XXX`。我没有发现是这样。

启动分页机制时，使用的是对等映射。我不理解“对等映射”是什么意思。

0x8000000，等于128M。书上的这段解释，我不理解。但我能明白，书上采取了什么方法。

不能由编译器随意安排可执行文件的`p_vaddr`，由两种方法，一，修改分页机制中的配置。（不理解）。

二，使用`ld`的选项`Ttext`，指定生成的可执行文件的`e_entry`。具体命令如下：

```shell
ld -s -Ttext 0x30400 -o kernel.bin kernel.asm
```

e_entry，程序的入口地址，30400h。属于elf_header。

p_vaddr，段的第一个字节在内存中的虚拟地址。属于program header。

### elf header

| 项目        | 值    | 说明                                 |
| ----------- | ----- | ------------------------------------ |
| e_ident     |       |                                      |
| e_type      | 2H    | 可执行文件                           |
| e_machine   | 3H    | 80386                                |
| e_version   | 1H    |                                      |
| e_entry     | 3040H | 入口地址                             |
| e_phoff     | 34H   | Program header table在文件中的偏移量 |
| e_shoff     | 448H  | section header table在文件中的偏移量 |
| e_flags     | 0H    |                                      |
| e_phsize    | 34H   | ELF header的大小                     |
| e_phentsize | 20H   | 每一个program header大小是20H字节    |
| e_phnum     | 1H    | Program header table中只有1个条目    |
| e_shentsize | 28H   | 每个section header的大小是28H个字节  |
| e_shnum     | 4H    | Section header table中有4H个条目     |
| e_shstrndx  | 3H    | 包含节名称的字符串表是第3个节        |

### program header

| 项目     | 值     | 说明                                                       |
| -------- | ------ | ---------------------------------------------------------- |
| p_type   | 1H     | PT_LOAD                                                    |
| p_offset | 0H     | 段的第一个字节在文件中的偏移                               |
| p_vaddr  | 30000H | 段的第一个字节在内存中的虚拟地址                           |
| p_paddr  | 30000H | ？                                                         |
| p_filesz | 40DH   | 段在文件中的长度                                           |
| p_memsz  | 40DH   | 段在内存中的长度。不管在哪里，长度还是那个长度，应该相同。 |
| p_flags  | 5H     | ？                                                         |
| p_align  | 1000H  | 啥意思？                                                   |



成员太多了，十分钟内，我记不住，好像也没必要记住。

应该把文件从开头40DH字节的内容放到内存0x30000处。而程序的入口地址是0x30400处，所以，程序实际上只有0x3040D - 0x30400 + 1 = D + 1 字节。

上面的两个表格，elf header 和 program header，意义仅仅是“说明”和“项目”这两列，告诉我，每个字段表示的含义。而我却被误导，每个字段的值是固定的。这导致我理解不了代码中对 memcpy 的参数赋值。memcpy 的参数赋值，与表格中的字段的固定值对应不上。

能帮助我理解调用 memcpy 的资料是：

### elf header

```c
typedef struct{
  unsigned char 												e_ident[EI_NIDENT];
  Elf32_Half														e_type;
  Elf32_Half														e_machine;
}Elf32_Ehdr;
```

太长了，懒得全部抄写完。

又一个重要表格。

| 名称          | 大小 | 对齐 | 用途               |
| ------------- | ---- | ---- | ------------------ |
| Elf32_Addr    | 4    | 4    | 无符号程序地址     |
| Elf32_Half    | 2    | 2    | 无符号中等大小整数 |
| Elf32_Off     | 4    | 4    | 无符号偏移         |
| Elf32_Sword   | 4    | 4    | 有符号大整数       |
| Elf32_Word    | 4    | 4    | 无符号大整数       |
| unsigned char | 1    | 1    | 无符号小整数       |



这个表格中的“大小”这一栏，再加上 struct Elf32_Ehdr 中每个成员的排列顺序，就能理解 memcpy 中的参数是怎么计算出来的。举两个例子。

```assembly
xor   esi, esi
mov   cx, word [BaseOfKernelFilePhyAddr+2Ch];`. ecx <- pELFHdr->e_phnum
```

BaseOfKernelFilePhyAddr 是内核的物理地址，BaseOfKernelFilePhyAddr+2Ch 又是什么呢？

从内核文件的开始处偏移 2CH 个字节，刚好对应 struct Elf32_Ehdr 的 e_phnum 成员。计算一下，是不是这样。

下面是在centos8上查看elf的源码。

```shell
[root@localhost pegasus-os]# ls /usr/include/elf.h
/usr/include/elf.h
```



```c
// 在 /usr/include/elf.h 中用关键词 Program 搜索
/* Program segment header.  */

typedef struct
{
  Elf32_Word    p_type;                 /* Segment type */
  Elf32_Off     p_offset;               /* Segment file offset */
  Elf32_Addr    p_vaddr;                /* Segment virtual address */
  Elf32_Addr    p_paddr;                /* Segment physical address */
  Elf32_Word    p_filesz;               /* Segment size in file */
  Elf32_Word    p_memsz;                /* Segment size in memory */
  Elf32_Word    p_flags;                /* Segment flags */
  Elf32_Word    p_align;                /* Segment alignment */
} Elf32_Phdr;
```



```c
typedef struct
{
  unsigned char e_ident[EI_NIDENT];     /* Magic number and other info */
  Elf32_Half    e_type;                 /* Object file type */
  Elf32_Half    e_machine;              /* Architecture */
  Elf32_Word    e_version;              /* Object file version */
  Elf32_Addr    e_entry;                /* Entry point virtual address */
  Elf32_Off     e_phoff;                /* Program header table file offset */
  Elf32_Off     e_shoff;                /* Section header table file offset */
  Elf32_Word    e_flags;                /* Processor-specific flags */
  Elf32_Half    e_ehsize;               /* ELF header size in bytes */
  Elf32_Half    e_phentsize;            /* Program header table entry size */
  Elf32_Half    e_phnum;                /* Program header table entry count */
  Elf32_Half    e_shentsize;            /* Section header table entry size */
  Elf32_Half    e_shnum;                /* Section header table entry count */
  Elf32_Half    e_shstrndx;             /* Section header string table index */
} Elf32_Ehdr;
```



在 e_phnum 的前面，有4个 Elf32_Half  类型的成员（8个字节）、2个 Elf32_Word 类型的成员（8个字节）、1个 Elf32_Addr 类型的成员（4个字节）、2个 Elf32_Off 类型成员（8个字节），总计占用 8 + 8 + 4 + ~~16~~ = 36。这个计算结果，仍然与代码对应不上。（耗费了不少时间。这类小问题，我若不能准确地心算出来，那就直接用笔算吧）

原来，我漏掉了开头的 `unsigned char e_ident[EI_NIDENT]; `，8个字节，36 + 8 = 44 = 32 + 12 = 2CH。与代码中的 BaseOfKernelFilePhyAddr+2Ch 吻合。仅这项只需大概2分钟就能完成的理解，居然耗费48分钟，原因是我漏掉了开头的那个成员变量！不经意间的遗漏，影响很恶劣。

为什么从内核文件的开头偏移若干个字节就能找到某个struct的成员结构？因为，C语言中的struct，在二进制数据中的本质就是一段连续的内存区域（至少逻辑上是这样的）。在这段内存区域中，又分成若干小段内存区域，每个内存区域是这个struct的一个成员变量。

上面这段话是正确的呢？为啥是这样的呢？是正确的，我好不容易才在C语言和汇编代码对同一个东西的表示形式中得到的认识，并且，通过查看二进制数据验证了这个认识。为啥是这样？不知道，知道了也没价值，经过查看二进制数据，它就是这样的！

e_phoff。从开头开始，1个char(8个字节)，2个Elf32_Half（4个字节），1个Elf32_Word（4个字节），1个Elf32_Addr（4个字节），总计 8 + 4 + 4 + 4 = 20 个字节，与1CH = 28个字节对应不上。

<u>上面的理解，是错误的，错误点是 `unsigned char e_ident[EI_NIDENT]; ` 。这个成员变量的长度不是8个字节，而是16个字节。</u>

怎么发现这个错误的？停顿之后，再看一眼，就发现了。这是对C语言中的数组概念理解错了。一个char是就是一个字节，一个char数组占用的空间，是这个char数组的元素个数个字节。

所以，

e_phoff 的偏移量是 16 + 4 + 4 + 4 = 28 = 1CH 个字节；

e_phnum的偏移量是 16 + 2 + 2 + 4 + 4 + 4+ 4 + 4 + 2 + 2 = 44  = 32 + 12 = 2CH 个字节；

分析完了 memcpy 的一个参数，其他的参数用同样的方法分析，不再赘言。

已经理解了 `mempcpy(p_vaddr, 段在文件中的偏移, 段的长度)`。再讲一讲，将内核在内存中重新放置是怎么回事。

内核被loader读取到内存中后，存储在内存中的某片区域。但是执行内核时，不是从这片内存中取指令进行执行（为啥不如此？我不知道。应该也可以这样做）。把内核文件中的程序段，从内存区域读取到规划好的内核区域。完成这个工作的，就是 memcpy 函数。

这个函数，有三个参数，第一个参数，是程序段在内存中（将来执行内核指令时，就从这片内存读取指令）的虚拟地址。第二个参数，是程序段在文件中的偏移量。在文件中的偏移量也是在目前存放内核文件的内存中的偏移量。无论数据在文件中还是在内存中，文件块距离文件开头的偏移量是不会变化的。第三个参数，是程序段的长度。从偏移量即程序段的开始位置开始，复制X长度的数据到内存的某个地址。这就是 memcpy 所做的事情。

使用汇编代码完成重新放置内核的功能，主要内容就是调用 mempcy，难点是获取它的三个参数，并且将参数传递给 memcpy。

使用栈传递参数。入栈顺序与 memcpy 的参数顺序（从左到右）相反。第一个入栈的是段的长度，第二个入栈的是段在文件中的偏移量，第三个入栈的是段在内存中的虚拟地址。

从ELF header开头偏移2CH个字节，就是程序头的个数。偏移1CH个字节，是第一个程序头在文件中的偏移量OFF。文件开头位置 + OFF就是第一个程序头的开始位置。每个程序头的长度是32个字节，所以下一个程序头（若存在下一个程序头）的开始位置是上一个程序头的开始位置 + 32。从程序头中获取这个程序头描述的程序段的信息：在内存中的虚拟地址、在文件中的偏移量、这个段的长度。

获取这些信息，依靠两块知识：elf header 结构 和 program header。

### memcpy

怎么实现这个函数？怎么把数据从一个位置复制到另外一个位置。我认为，代码应该是下面这样的：

```assembly
mov eax, [posA]
mov [posB], eax
```



作者的实现：

```assembly
; ------------------------------------------------------------------------
; 内存拷贝，仿 memcpy
; ------------------------------------------------------------------------
; void* MemCpy(void* es:pDest, void* ds:pSrc, int iSize);
; ------------------------------------------------------------------------
MemCpy:
	push	ebp
	mov	ebp, esp
	
	; 因为会改变它们，所以需要保存原来的值
	push	esi
	push	edi
	push	ecx
	
	; 栈地址是从高向低变化的，先入栈的数据在高地址，后入栈的数据在低地址
	mov	edi, [ebp + 8]	; Destination
	mov	esi, [ebp + 12]	; Source
	mov	ecx, [ebp + 16]	; Counter
.1:
	cmp	ecx, 0		; 判断计数器
	jz	.2		; 计数器为零时跳出			; 使用 jz，写boot.asm时，在这个系列（比较）指令的使用上栽过几个大跟头。

	mov	al, [ds:esi]		; ┓
	inc	esi			; ┃
					; ┣ 逐字节移动
	mov	byte [es:edi], al	; ┃
	inc	edi			; ┛

	dec	ecx		; 计数器减一
	jmp	.1		; 循环
.2:
	mov	eax, [ebp + 8]	; 返回值。函数的返回值放在eax中，这是约定。

	pop	ecx
	pop	edi
	pop	esi
	; 在开头时入栈，结束时就应该出栈
	; 这句，非必须，因为esp在整个函数中没有被改变过
	mov	esp, ebp
	pop	ebp
	
	; 不能缺少。调用函数的地方
	ret			; 函数结束，返回
; MemCpy 结束-------------------------------------------------------------
```



在这个函数中，唯一不理解的是，将 `[ds:esi]` 处的数据复制到 `[es:edi]` 处。内核文件放置在数据段吗？很好验证。能在加载内核文件的代码中找到答案。

详细资料

https://www.cnblogs.com/feng9exe/p/6899351.html

## 跳入保护模式

定义三个全局描述符，分别是：0~4GB的可执行段、0~4GB的可读写段和一个指向显存的段。

还要定义这三个全局描述符对应的三个选择子。选择子就是全局描述符在GDT中的偏移量，所以，直接用全局描述符的标号减去第一个全局描述符（空全局描述符，表示GDT的初始位置）。

全局描述符的结构比较复杂，我没有记住。它由段地址、段界限和段属性组成。段属性中有一个值决定段界限的单位是字节还是KB。

在前几章的代码中，创建全局描述符时，段地址是0。然后，修改全局描述符中的存储段地址的数据，达到修改全局描述符的目的。在本章，并没有这样做。因为，段描述符的段在被创建时，段地址为0，此后，这个段的段地址就一直是0，并没有修改。



计算标号（变量）的物理地址的公式是：
$$
标号（变量）的物理地址 = BaseOfLoader * 10h + 标号（变量）的偏移量
$$

### 进入保护模式的步骤

#### 关闭软驱马达

不关闭软驱马达，软驱指示灯会一直亮着。我觉得，关闭它不是进入保护模式的必要条件，一直亮着就让它亮着吧。

#### 关闭中断

```assembly
cli
```

为啥要关闭中断？

#### 打开A20地址线

A20什么？究竟是不是地址线？是，就叫地址总线。

早期，总线是20位的，能表示的最大地址是2的20次方。这个说法，有问题。2的20次方，这个数字的低20位全是0，那么，这个数字要么是0，要么至少是21位。前面的两种说法，都有问题。总线是20位的，能表示的最大地址是这样一个数字M：这个数字有20位，每位都是1，20个1，值是2的20次方-1。
$$
1111 1111 1111 11111 1111
$$
当交给总线的地址N大于M时，地址总线会对N进行处理，用下面的公式：
$$
A = N - M
$$
用简单的例子来理解。总线能接受的最大地址是10，当传递给它的地址是8时，总线不做任何处理。当传递给它的地址是12时，总线会对地址进行处理，12 - 10 = 2，2才是总线真正接收并传递给其他硬件的地址。

那么，内存地址会大于20位的地址总线所能处理的最大地址M吗？内存地址的计算公式是：
$$
内存地址 = 16位段基址 << 4 + 16位段偏移量 = 16位段基址 * 16 + 16位段偏移量
$$
假如16位段基址左移4位后是	`1111 1111 1111 1111 0000`，段偏移量是	`1111 1111 1111 1111` ，二者相加，结果大于M。所以，16位CPU能产生的地址可能会大于20位总线能处理的最大地址M，20位CPU更是如此。为了防止CPU产生20位总线所不能处理的内存地址，底层工程师设计了一个“回卷”机制，就是上文提到的那个对大于地址总线所能处理的最大地址的处理公式。

当数据总线扩展到32位后，大于20位的内存地址也能够被处理了，但数据总线默认是打开了“回卷”机制的（兼容20位地址总线）。要使用32位总线的全部地址，需要关闭“回卷”机制。

> 这个知识没啥用。

汇编代码是：

```assembly
in al, 92h
or al, 00000010b
out 92h, al
```



#### 设置cr0

```assembly
mov eax, cr0
or cr0, 1
mov cr0, eax
```



#### 跳入保护模式



## 向内核交出控制权

只需要一个跳转指令 `jmp` 。然而，这个跳转涉及到段和GDT。我还没有重新读GDT这块知识。我没有体会到GDT是必要的，想先不使用GDT实现执行内核指令。

## 其他

### FAT12

### 摘要

文件系统是一种软件，安装在存储设备上，例如软盘。FAT12就是一种文件系统，容量较小。虽然被淘汰了，但它结构简单，能通过了解它对文件系统是什么东西有一个感性认识。

#### 结构

FAT是一种过时的文件系统。它的设计方式很简单，学习起来很容易。通过了解这个简单的文件系统，为掌握复杂的文件系统打下基础。

FAT12的结构是这样的，由下面几种元素构成：

引导扇区。FAT12这种文件系统的头信息存储在这个扇区。哪些头信息呢？比如，根目录占用多少个扇区，每个FAT占用多少个扇区。

两个FAT。第0个、第1个FAT项不使用，从第3个FAT项开始，每个FAT项的编号表示文件所占用的扇区在数据区的扇区编号。两个FAT的内容相同，第2个FAT是第1个FAT项的备份，也可以叫冗余。

根目录区。根目录区由若干个结构组成。每个结构的长度是32字节。这种结构叫根目录项。每个根目录项对应一个文件。在使用FAT12文件系统的存储设备上，有多少个文件就有多少个根目录项。

数据区。不用解释，就是存储文件的区域。其他区域，比如FAT、根目录区，存储的不是文件的数据，而是文件的元数据。

#### 查找文件

在使用FAT12文件系统的存储设备，例如软盘，查找文件，过程是这样的。

遍历根目录区。根目录区由若干个根目录项组成。每个根目录项中包含文件的文件名和在数据区的第一个扇区的编号。假如要查找的目标文件是"LOADER BIN"，那么，遍历根目录区，检查根目录项的文件名是否为"LOADER BIN"。若是，当前根目录项就是目标文件对应的根目录。在这个根目录偏移0X1A的位置，就是目标文件在数据区的第一个扇区的扇区号，同时，也是目标文件在FAT中的第一个FAT项的编号。

FAT表实质是一个单链表。每个FAT项的值是下一个FAT项的编号。当FAT项的值是A(忘记了具体数字)时，表示当前FAT项是该文件的最后一个FAT项。当FAT项的值是B（忘记了具体值），表示当前FAT项所表示的扇区是一个坏扇区，即不能被正确读取。

#### 其他

软盘已经被淘汰，想找一张软盘来玩很不容易。但可以使用 bximage 创建一个虚拟软盘 a.img ，然后通过下面的命令，把数据写入软盘中。bximage 是bochs附带的工具，能创建虚拟软盘，还能创建虚拟硬盘。那bochs又是什么呢？哈哈，像套娃一样。bochs是一种功能很强大的虚拟机，可以断点调试，看到所有寄存器（通用寄存器和段寄存器）和堆栈的值。与更常用的vmware不同。

```shell
dd if=boot.bin of=a.img count=1 conv=truc
sudo mount -o loop /mnt/floppy
cp ./loader.bin /mnt/floppy
sudo umount -o /mnt/floppy
```



### 将内核指令重新放置到内存

#### 摘要

操作系统的内核是一个elf文件。加载内核，需要从存在于内存中的内核文件数据中读取所有的程序段，并把这些程序段复制到规划好的内存位置（内核指令应该占据的内存位置），然后将CPU的控制权移交给这些内核指令。操作系统就正式运行起来了。

#### 把内核放入内存，究竟需做什么

写满实现内核功能的代码的文件会被编译成一个ELF文件。这个ELF文件不同于LOADER BIN文件。后者实质是一个没有使用DOS命令的COM文件。因此，只需将它原封不动地从存储设备读入到内存中，然后跳转到这个内存区域的开始，就将CPU的控制权交给了LOADER。

ELF文件是当前Linux系统上的可执行文件格式。写一个C程序，然后编译成可执行文件，使用 file 查看这个文件，能看到这个文件是ELF文件。

ELF文件由program header table、elf header、section header table、section组成。只有elf header的位置是固定的，在elf文件的开始位置。其他几个成员的位置不固定。

将内核指令重新放置到内存中，需做两件事情：一、把内核文件读入内存中。二、把内存中的内核的程序段全部复制到规划好的内存位置。

第一件事情，熟悉FAT12文件系统，就能做到。我已经独立写代码完成了这个功能并通过了测试。在本文不想再赘言。

第二件事情，要想完成它，需了解elf结构，需知道如何把数据从内存A位置复制到内存B位置，也就是说，需要实现一个函数，memcpy(int dest, int off, int size)。三个参数分别是：在内存中的虚拟地址、程序段在文件中的偏移量、程序段的长度。三个参数都能从ELF文件的elf头和程序头中获取。

参照位置是elf文件的开头。偏移量28个字节的内存位置，给它起个标记叫P，从P开始的若干个字节（忘记了具体数字）的内存存储的是程序段的偏移量。文件开头加上这个偏移量，是第一个程序头的内存初始位置。

程序头也存储在一片内存中。用C语言中的struct帮助描述。程序头是一个struct结构，成员变量有程序段的长度、程序段的偏移量、程序段在内存中的虚拟位置（也就是这个程序段将要被重新放置在内存中的位置）。这三个成员变量，就是函数memcpy需要的三个参数。

#### memcpy的实现

memcpy，有三个参数，分别是：数据要被复制到的内存地址dst，数据的原始地址src，数据的长度size。这个函数的功能是，把src处的size个字节的数据复制到dst处，返回值是src。

直接上代码。

```assembly
mempcy:
	push ebp
	mov	ebp, esp
	
	push esi
	push edi
	push ecx
	
	mov esi, [ebp+12]			; src
	mov edi, [ebp+8]			; dst
	mov ecx, [ebp+16]			; size

.1
	cmp ecx, 0
	jz .2
	mov al, [ds:esi]
	mov [es:edi], al
	inc esi
	inc edi
	dec ecx
	jmp .1
.2
	mov eax, [ebp+8]
	
	pop ecx
	pop edi
	pop esi
	pop ebp
	
	ret
```



详细解读这个函数的实现。

在汇编中实现一个函数，模板是：

```assembly
functionName:
	; some code
	; some code
	
	ret
```

汇编函数必须用ret结尾。它的作用是在函数执行结束后，返回调用函数的上层代码的下一条指令。

调用函数时，使用栈传递参数给函数。在函数内部，获取参数时，再从栈中获取参数。

```assembly
mov esi, [ebp+12]			; src
mov edi, [ebp+8]			; dst
mov ecx, [ebp+16]			; size
```

ebp指向栈的开始位置栈顶，偏离栈顶4个字节的位置存储的是调用函数指令的下一条指令的位置（大概就是这个意思），它是执行 call 指令时入栈的，是函数调用过程中最后一个入栈的数据。在它之前依次是函数的第一个、第二个、第三个参数入栈（在本函数中），相对于栈顶的偏移量依次是8个字节、12个字节、16个字节。

由于上面的代码修改了esi、edi、ecx中的值，需要在修改之前将它们保存起来，在函数结束时再恢复它们原来的值，所以有下面的代码：

```assembly
push esi
push edi
push ecx

; some code
; some code

pop ecx
pop edi
pop esi
```

内存的最小单位是字节，本函数也按字节来复制数据，这不是必须的。复制数据使用

```assembly
mov al, [ds:esi]
mov [es:edi], al
```

eax存储2个字，4个字节；ax存储1个字，2个字节；al存储1个字节。ds是数据段，es是什么？有多少个字节，就需要重复多少次上面的复制操作。因此，需要一个循环。

```assembly
.1
	cmp ecx, 0
	jz .2
	mov al, [ds:esi]
	mov [es:edi], al
	inc esi
	inc edi
	dec ecx
	jmp .1
```

在汇编中，loop指令能实现循环功能，本函数却并未使用。这是为了规避恐怖的ecx陷阱。使用loop指令时，需ecx配合。在循环过程中，ecx的值会自动减少。当ecx的值是0时，循环结束。陷阱就出现在这里。具体是咋回事，我忘记了。但我遇到过，再加上在函数体中，可能会修改ecx。为了避免种种诡异的问题，本函数一般使用jmp指令，再配合手工递减的ecx来实现循环功能。作者的其他汇编代码都会如此，尽量不使用loop指令。





## 好资料

The Linux Information Project

http://www.linfo.org/index.html

