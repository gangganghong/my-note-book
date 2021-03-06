# 保护模式

## 认识保护模式

### GDT

#### 摘要

在操作系统中，全局描述符是什么？GDT又是什么？在进入保护模式之前，准备好GDT和GDT中的描述符是必须的吗？用汇编代码怎么创建描述符？本文解答上面几个问题。

在实模式下，CPU是16位的，意思是，寄存器是16位的，数组总线（data bus）是16位的，但地址总线是20位的。物理内存地址的计算公式是：
$$
物理地址 = 段地址 * 16 + 偏移量
$$
段地址和偏移量都是16位的，能寻址的最大内存地址是1M。

1M是怎么计算出来的？2的20次方就是1M，能表示的内存地址是 0~（2的20次方-1）。用简单例子来理解，1位十进制数能表示的最大数是10 - 1 = 9，但1位十进制数能表示的数却是 `0` ，和 `1-9` ，总计10个数字。

若一个内存地址是`20:30`，最终内存地址是：`20 * 16 + 30`。

在保护模式下，内存地址仍然用“段地址：偏移量”的方式来表示。不过，“段地址”的含义不同于实模式下的“段地址”。在实模式下，段地址是物理地址的高地址部分，具体说，是高16位部分。而在保护模式下，段地址是选择子，指向一个结构，这个结构描述了一个内存区域，告知该区域的内存地址从哪里开始，在哪里结束，还告知了这片内存能不能被访问、能不能被读取等数据。这个结构组成一个集合，叫GDT，而这个结构叫GDT项，它有一个术语，叫“描述符”。

GDT的作用是提供段式存储机制。段式存储机制由段寄存器和GDT共同提供。段寄存器提供段值，即描述符在GDT中的索引，也就是选择子。根据选择子在GDT中找到目标描述符。这个描述符中包含一段内存的初始地址、这段内存的最大地址、这段内存的属性。

#### GDT的构成

GDT项即全局描述符的长度是8个字节，64个bit，64个位，0~63位，而不是1~64位。下图是写了位编号的8个字节。真实的全局描述符是不折行的，这里无法在一行显示全部数据，因此折行了。

```shell
63|62|61|60|59|58|57|56| 55|54|53|52|51|50|49|48| 47|46|45|44|43|42|41|40| 39|38|37|36|35|34|33|32| 31|30|29|28|27|26|25|24| 23|22|21|20|19|18|17|16| 15|14|13|12|11|10|09|08| 07|06|05|04|03|02|01|00|
```



`15|14|13|12|11|10|09|08| 07|06|05|04|03|02|01|00|`。段界限1。段界限的 0~15 位。描述符的 0~15 位。



`39|38|37|36|35|34|33|32| 31|30|29|28|27|26|25|24| 23|22|21|20|19|18|17|16|`。段基址1。段基址的 0~23位。描述符的 16~39位。

`55|54|53|52|51|50|49|48| 47|46|45|44|43|42|41|40|`。很复杂，很碎片化，需进一步放大观察。

`43|42|41|40|`，TYPE。4位。

`44`，S。指示段是系统段(0)还是数据段(1)。1位。

`46|45`，DPL。2位。

`47`，P。1位。

上面是一个字节，下面是第二个字节。

`51|50|49|48|`。段界限2。段界限的第 16~19 位。描述符的第 48~51 位。段界限一共有20位。

`52`。AVL。1位。

`53`。0。1位。

`54`。D/B。1位。

`55`。G。1位。若G位是0，段界限的粒度是1字节。若G位是1，段界限的粒度是4KB。

段属性占用空间的位数：4 + 1 + 2 + 1 + 1 + 1 * 3 = 12。



`63|62|61|60|59|58|57|56|`。段基址2。段基址的第 24~31 位。描述符的第 56~63 位。段基址一共有32位。



描述符的结构比较复杂，要记住它，有点困难，不过并非不可能记住。作者觉得没有必要一个字节不差地背诵出来。

#### 选择子

描述符的选择子的长度是16位。

```shell
15|14|13|12|11|10|09|08| 07|06|05|04|03|02|01|00|
```



`01|00|`，RPL。

`02`，T1。

`15|14|13|12|11|10|09|08| 07|06|05|04|03`，描述符在GDT中的索引。



#### 段式存储机制的寻址方式

段地址存储的是描述符的选择子，根据选择子能找到GDT中对应的描述符。从描述符中获取段基址，然后加上段式存储机制中的偏移量，就是线性地址。在当前语境下，线性地址等同物理地址。

#### 概念比较

逻辑地址。段式机制的地址，例如“段地址：偏移量”，就是逻辑地址。

线性地址。在保护模式下，用逻辑地址中的段地址从GDT中找到描述符，然后从描述符中获取段的基址，段基址加上偏移量的结果就是线性地址。

如上文所言，线性地址目前可视为物理地址。开启分页机制后，线性地址不能等同于物理地址。物理地址是物理内存的一个编号。

#### 作者的疑问

进入保护模式前，为什么需要创建好描述符、选择子、GDT？这些是必要条件吗？

作者曾认为这些不是必须。再次了解段式存储机制后，改变了看法：进入保护模式前，必须准备好GDT、描述符和描述符选择子。这是由保护模式下的内存寻址方式决定的。

无论是在实模式下还是保护模式下，都需要使用内存。在保护模式下，怎么找到某片内存呢？保护模式下，使用段式机制。回忆一下，段式存储机制的寻址方式是：
$$
段地址（选择子）-----》在GDT中找到描述符----》在描述符中找到段基址----》段基址+偏移量 = 线性地址
$$
不在进入保护模式前准备好选择子、GDT、描述符，就无法在保护模式中使用内存。

作者还有一个疑问：上面的寻址过程是CPU自动完成的吗？

#### 实现描述符

##### C语言

###### 描述符

下面内容的前提是，32位CPU。

```c
struct {
	int segmentLimit1:16;															// 段界限1
  int segmentBaseAddress1:24;											// 段基址1
  char attributeType:4;														// 段属性，TYPE
  char attributeS:1;															// 段属性，S
  char attributeDPL:2;														// 段属性，DPL
  char attributeP:1;															// 段属性，P
  char segmentLimit2:4;														// 段界限2
  char attributeAVL:1;														// 段属性，AVL
  char attributeZero:1;														// 段属性，值为0
  char attributeDB:1;															// 段属性，DB
  char attributeG:1;															// 段属性，G
  char segmentBaseAddress2;												// 段基址2
}GlobalDescriptor;
```

上面的用法是错误的。对位域的使用是错的，换成int来使用位域也无能力写正确，因为太麻烦。在这个知识点耗费了不少时间。

参考书中代码后，写出下面的代码：

```c
struct{
  unsigned short segmentLimitLow;															// 段界限1，16位，0~15 位。描述符的第 0~15 位。
  unsigned short segmentBaseAddressLow;												// 段基址低16位，0~15 位。描述符的第 16~31 位。
  unsigned char	segmentBaseAddressMid;												// 段基址 16~23 位。描述符的第 32~39 位。
  unsigned char attribute;																		// 段属性。描述符的第 40~47 位。
  unsigned char segmentLimitHight_attribute2;				// 段界限 16~19 位，第 20~23 位是段属性。描述符的第 48~55 位。
  unsigned char	segmentBaseAddressHigh;												// 段基址 24~31 位。描述符的第 46~63 位。			
}GlobalDescriptor;
```

段基址虽然存储在描述符的第 16~39 位 和第 24~31 位两段连续的空间中，但用C语言表示它的时候，却人为地将它拆分成了“低位”、“中位”和“高位”三部分，也就是，把描述符的第 16~39 位拆分成了第 16~31 位和第 32~39 位两段。在C语言中，没有现成的能存储23位的整数类型，却用能存储16位和8位的整数类型。将段基址连在一起的24位拆分，用C语言表示更方便。

###### C语言中的位域

前面已经用到了位域，那就简单学习一下位域的知识吧。

用两段代码开始。

```c
struct{
	unsigned int age;
	unsigned int height;
}Person;
```



```c
struct{
	unsigned int age:3;
  unsigned int height:4;
}Person2;
```



第二段代码使用了位域，第一段代码是普通的`struct`结构。位域语法与`struct`的差异仅在于声明成员变量的语法不同。

`struct`结构中，声明成员变量的语法是`unsigned int age`。在位域中，声明成员变量的语法是`unsigned int age:3`。后者指定了成员变量使用的bit的数量，是3个，而不是1个字节、8个bit。

第一段代码创建的`Person`占用8个字节，第二段代码创建的`Person2`占用4个字节。

抽象出位域的成员变量的声明语法：`dataType VariableName:bitCount`。`dataType`只能是`int`系列的整数类型，即只能是`int`、`unsigned int` 和 `signed int` 三种类型，不能是`char`等类型。这是语法规定。`bitCount`不能超过8个字节。

##### nasm汇编

用汇编语言表示描述符，是作者写本文的终极目的，前面的一切都是铺垫和基础。C语言表示描述符，在前面写出来，是因为它是作者理解描述符的汇编代码的大功臣。作者在看描述符的汇编代码前，没有学过汇编语言，所以第一次看描述符的汇编代码时，怎么都理解不了。看了别人写的描述符的C语言代码后，才恍然大悟，突然理解了描述符的的汇编代码。

所以，在前文给出描述符的C代码，一是为了纪念这个大功臣，二是让与曾经看不懂汇编代码的作者一样的读者也能借住C代码理解汇编代码。当然，可能是作者多虑了，读者朋友才不会像作者这么愚钝呢。

###### 不使用宏

第一个问题，创建一个描述符，例如`DESC_VIDEO`，语法是什么样的。

第二个问题。描述符的实质是段基址、段界限和段属性。是直接用代码堆砌出描述符呢还是根据给定的段基址、段界限和段属性经过运算拼凑出描述符？

先解答第二个问题。直接用代码堆砌出描述符的汇编代码如下：

```assembly
DESC_VIDEO	dw	3120h																											; 描述符的第 0~15 位
	dw	111Fh																																; 描述符的第 16~31 位
  db	EFh																																	; 描述符的第 32~39 位
  db	42h																																	; 描述符的第 40~47 位
  db	00h																																	; 描述符的第 48~55 位
  db	FFh																																	; 描述符的第 56~63 位
```

与前面的C代码比较，每行对应一个struct的成员变量。从上面的汇编代码能看出段基址、段界限和段属性是什么吗？看不出来，需要计算。而且，总不能拿到给定的段基址、段界限和段属性后，先将它们转换成二进制，然后再分割填到上面的代码中吧？最好是给定段基址、段界限和段属性后，经过一段代码处理，就自动构建了描述符。这就是下面要写的方式。

###### 宏

汇编中的宏类似C语言中的函数，给定参数，函数会完成一些功能。这个宏接收段基址、段界限和段属性，然后生成描述符。

宏的语法是什么样的？创建一个宏的模板是：

```assembly
%macro macroName paramCount
	;some code
	;some code
%endmacro
```

创建描述符的宏是：

```assembly
; 三个参数依次是：base(段基址)、limit（段界限）、attribute（段属性）
; 在宏中需用到这三个参数时，对应的代号分别是：%1、%2、%3。
; base--32位，limit--20位，attribute--12位
%macro Descriptor	3
	dw	%2 & FFFFh															; 段界限的第 0~15 位。16位
	dw	%1 & FFFFh															; 段基址的第 0~15 位。16位。
	db	(%1 & FF0000h) >> 16										; 段基址的第 16~23 位。8位。
	db	%3 & FFh																; 段属性的第 0~7 位。48位。
	db	(%2 & F0000) | (%3 >> 8) 								; 段界限的第 16~20 位 和 段属性的第 8~11 位。56位。
	db	%1 >> 24																; 段基址的第 24~31 位。8位。
%endmacro
```

使用这个宏创建一个描述符，代码如下：

```assembly
DESC_VIDEO:	Descriptor	0B8000h		0ffff			0
```

段属性是随便设置的。描述符的段属性比较复杂。作者暂时没有弄清楚。

### 实模式到保护模式

#### gdtr

gdtr是一个寄存器，它的结构如下所示：

```shell
47|46|45|44|43|42|41|40|39|38|37|36|35|34|33|32|31|30|29|28|27|26|25|24|23|22|21|20|19|18|17|16| 15|14|13|12|11|10|09|08|07|06|05|04|03|02|01|00|
```

第`15~0`位，是界限。

第`47~16`位，是基地址。

gdtr存储的是GDT的物理地址。将GDT的物理地址存储到gdtr寄存器的语法是：

```assembly
lgdt	[GdtPtr]	; GdtPtr是GDT物理地址的标号，或者叫变量
```



### 描述符属性

1. S位是0时，系统段。S位是1时，数据段。供硬件使用的数据是属于系统的，供软件使用的是属于数据的。各种“门”结构，例如任务门、调用门等，是系统段。

2. G位指示段界限大小的粒度。当G位是0时，粒度是字节；当G位是1时，粒度是4KB。

3. TYPE，共4位，表示内存段或门的子类型。
   1. 系统段
      1. `0010`，表示是局部描述符表。
   2. 非系统段
      1. 代码段
         1. 4位依次是：`XRCA`。
            1. X，EXecutable，表示该段是否可执行。数据段是不可执行的，X为0；代码段是可执行的，X为1。
            2. R。R为1，可读；R为0，不可读。
            3. C，Conforming。C为1，一致性代码段；C为0，非一致性代码段。很复杂，没弄懂。
            4. `A`，Accessed，由CPU来设置。当该段被CPU访问后，CPU就将`A`置1。
         2. `100*`，只执行代码段。
         3. `110*`，可执行、可读代码段。
         4. `101*`，可执行、一致性代码段。
         5. `111*`，可执行、可读、一致性代码段。
      2. 数据段。每个位的字母表示，看不出是什么意思。
         1. 4位依次是：`XWEA`。
            1. `X`，表示该段是否可执行。
            2. `W`，段是否可写。Writable。W为1，可写，数据段。W为0，不可写，代码段。
            3. `E`，标识段的扩展方向。Extend。E为0，表示向上扩展，地址递增，用于代码段和数据段。E为1，向下扩展，地址递减，用于栈段。
            4. `A`，Accessed，由CPU来设置。当该段被CPU访问后，CPU就将`A`置1。
         2. `000*`，只读数据段。
         3. `010*`，可读写数据段。
         4. `001*`，只读、向下扩展的数据段。
         5. `011*`，可读写、向下扩展的数据段。
4. DPL，Descriptor Privilege Level，描述符等级。
5. P，Present，段是否存在。若段存在于内存中，P为1；若段不存在于内存中，P为0。CPU只负责检查P字段。当P字段是0时，CPU会抛出异常。异常程序需要程序员处理。
6. AVL，AVaiLable。操作系统可随意用这位。
7. L。L为1，表示该段是64位代码段；L为0，表示该段是32位代码段。
8. D/B。指示有效地址（段内偏移地址）和操作数的大小。
   1. 对代码段来说，此位是D位。若D为0，表示指令中的有效地址和操作数是16位，指令有效地址用IP寄存器。若D为1，表示指令中的有效地址和操作数是32位，指令有效地址用EIP。
   2. 对栈段来说，此位是B位。若B为0，使用SP寄存器。若B为1，使用ESP寄存器。前者，栈的起始地址是16位寄存器的最大寻址范围，0xFFFF。后者，栈的起始地址是32位寄存器的最大寻址范围，0xFFFFFFFF。
9. G，Granularity，粒度，指定段界限的大小。若G为0，段界限的粒度是1字节。若G为1，段界限的粒度是4KB。

## 保护模式进阶





