# 《x86汇编语言：从实模式到保护模式》阅读笔记



一个字含有2个字节和16个比特。

一个双字含有4个字节、2个字和32个bit。

二进制数1000 0000中，位7的那个比特是“1”，也就是第8位。它是最高位。

一个存储器的容量是16个字节，2个字，地址范围为 (01 -- 02) ~~ () 。用存储器保存字数据时，可存放（2）个字，这些字的地址分别是（01、02），保存双字呢？保存一个双字。

检测点2.2

A3D8

低端字节序：高字节位于高地址部分，低字节位于低地址部分。

什么叫高字节？

高地址与低地址好区分，高字节和低字节，不知道是怎么区分的。

![image-20201222004306711](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201222004306711.png)

理解红线处，浪费了很多时间。如此简单的问题，我理解起来都费劲。

妨碍我正确理解这个问题的点是：我给“段地址+偏移地址”加了个假设，段地址是16位的，偏移地址是16位的，能表示的地址的最大值是2的16次方乘以2。这不是64KB。

正确的理解，应该是：段地址必须小于FFFFH，段地址+最大偏移量，必须小于FFFFH。内存地址必须在合法范围内。

![image-20201222020132094](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201222020132094.png)

两种特殊情况：

1. 段最大，即偏移地址的最大值最大，64KB，共16个段。
2. 段最小，即偏移地址的最大值取最小值，16B，共2的16次方个段。
   1. 最小值能不能是15B？好像可以，但是浪费空间，不能将1MB内存划分为数个段。
3. 理解这个，我又花了一些时间，时间久了，我又会忘记。
4. 记住，能使用的物理内存，由总线和寄存器功能决定。用两个寄存器表示的内存地址，必须在总线所能保存的地址的范围内。

![image-20201222021037322](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201222021037322.png)

为啥要按16字节对齐？是必须的吗？

检测点2.3

1. 8。

2. A、C、D。

3. ~~没有正确答案。~~两种情况：第一：段地址+偏移地址；第二：段地址*16 + 偏移地址。~~两种情况，我都没发现选项。~~

   按第二个规则计算，左移动4位，因为是16进制，只需要移动一位。有多个答案，我就不计算了。

习题

1. 65536.
2. [25BCH,1024*1024-25BCH]

## 进入保护模式

### 全局描述符表

11.2

![image-20201222023347729](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201222023347729.png)

GDT必须在进入保护模式前定义，所以，GDT通常定义在1M内存范围内。可以在进入保护模式后重新定义。

![image-20201222043854244](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201222043854244.png)

这张图来源于《操作系统真相还原》4.3节。

始终理解不了这段代码：

```assembly
; 宏 ------------------------------------------------------------------------------------------------------
;
; 描述符
; usage: Descriptor Base, Limit, Attr
;        Base:  dd
;        Limit: dd (low 20 bits available)
;        Attr:  dw (lower 4 bits of higher byte are always 0)
%macro Descriptor 3
	dw	%2 & 0FFFFh				; 段界限1
	dw	%1 & 0FFFFh				; 段基址1
	db	(%1 >> 16) & 0FFh			; 段基址2
	dw	((%2 >> 8) & 0F00h) | (%3 & 0F0FFh)	; 属性1 + 段界限2 + 属性2
	db	(%1 >> 24) & 0FFh			; 段基址3
%endmacro ; 共 8 字节

[SECTION .gdt]
; GDT
;                              段基址,      段界限     , 属性
LABEL_GDT:	   Descriptor       0,                0, 0           ; 空描述符
LABEL_DESC_CODE32: Descriptor       0, SegCode32Len - 1, DA_C + DA_32; 非一致代码段
LABEL_DESC_VIDEO:  Descriptor 0B8000h,           0ffffh, DA_DRW	     ; 显存首地址
; GDT 结束
```

段描述符，是否只根据硬件说明符（或其他某种要求）如此写代码就可以？段描述符宏的参数是根据什么给出的？

仅仅看宏的构成，我都看不明白。只能对照其他代码，来把握段描述符的实质了。

## GDT

极好的资料

https://www.cnblogs.com/iBinary/p/13198073.html

https://zhuanlan.zhihu.com/p/25867829

表示GDT的代码，C语言，

```c
// base: 基址（注意，base的byte是分散开的）
// limit: 寻址最大范围 tells the maximum addressable unit
// flags: 标志位 见上面的AC_AC等
// access: 访问权限
struct gdt_entry {
    uint16_t limit_low;       
    uint16_t base_low;
    uint8_t base_middle;
    uint8_t access;
    unsigned int limit_high: 4;
    unsigned int flags: 4;
    uint8_t base_high;
} __attribute__((packed));
```

与这张图，严格一一对应。

疑问：struct成员是按照定义成员的顺序在内存中排列的吗？即，成员在内存中的排列顺序和成员在定义中的顺序是严格一一对应的吗？

![img](https://pic2.zhimg.com/80/v2-7d368769d159149aa35949405a91d08d_1440w.png)

用下面的程序验证，答案是：是严格一一对应的。

```c
#include <stdio.h>
#include <stdlib.h>

struct gdt_entry {
    uint16_t limit_low;
    uint16_t base_low;
    uint8_t base_middle;
    uint8_t access;
    unsigned int limit_high: 4;
    unsigned int flags: 4;
    uint8_t base_high;
};
int main(int argc, char **argv)
{

 struct gdt_entry *desp = (struct gdt_entry *)malloc(sizeof(struct gdt_entry));
 desp->limit_low = 2;
 desp->base_low = 3;
 desp->base_middle = 4;
 desp->access = 10;
 desp->limit_high = 8;
 desp->flags = 7;
 desp->base_high = 5;

 printf("limit_low:%d\n", desp->limit_low);
 printf("limit_low address : %02X\n", &(desp->limit_low));
 printf("base_low address : %02X\n", &(desp->base_low));
 printf("base_middle address : %02X\n", &(desp->base_middle));
 return 0;
}
```

编译时有警告，但是，能正确指向。

疑问：

1. 如何打印内存地址而不出现警告？
2. 如何打印整型变量的内存地址？

```shell
chugangdeMacBook-Pro:c-example cg$ gcc -g -o struct-demo struct-demo.c
struct-demo.c:26:39: warning: format specifies type 'unsigned int' but the argument has type 'uint16_t *' (aka 'unsigned short *')
      [-Wformat]
 printf("limit_low address : %02X\n", &(desp->limit_low));
                             ~~~~     ^~~~~~~~~~~~~~~~~~
struct-demo.c:27:38: warning: format specifies type 'unsigned int' but the argument has type 'uint16_t *' (aka 'unsigned short *')
      [-Wformat]
 printf("base_low address : %02X\n", &(desp->base_low));
                            ~~~~     ^~~~~~~~~~~~~~~~~
struct-demo.c:28:41: warning: format specifies type 'unsigned int' but the argument has type 'uint8_t *' (aka 'unsigned char *') [-Wformat]
 printf("base_middle address : %02X\n", &(desp->base_middle));
                               ~~~~     ^~~~~~~~~~~~~~~~~~~~
                               %2s
```

多次执行，结果如下：

```shell
chugangdeMacBook-Pro:c-example cg$ ./struct-demo
limit_low:2
limit_low address : 4DE05A20
base_low address : 4DE05A22
base_middle address : 4DE05A24
chugangdeMacBook-Pro:c-example cg$ ./struct-demo
limit_low:2
limit_low address : EE405A80
base_low address : EE405A82
base_middle address : EE405A84
chugangdeMacBook-Pro:c-example cg$ ./struct-demo
limit_low:2
limit_low address : 87405A80
base_low address : 87405A82
base_middle address : 87405A84
chugangdeMacBook-Pro:c-example cg$
```

`limit_low address : 4DE05A20` 与 `base_low address : 4DE05A22` 地址差距是2个字节，即16位，刚好是一个 `uint16_t` 类型数据所占空间。

从上面得到的启发，GDT内的描述符如何构造？

按要求配置0到63位的数据，每一段，都有特定含义，能完成这个配置，就可以了。

再理解段描述符的汇编代码。

```assembly
; 宏 ------------------------------------------------------------------------------------------------------
;
; 描述符
; usage: Descriptor Base, Limit, Attr
;        Base:  dd
;        Limit: dd (low 20 bits available)
;        Attr:  dw (lower 4 bits of higher byte are always 0)
%macro Descriptor 3
	dw	%2 & 0FFFFh				; 段界限1，一个字，2个字节，16位
	dw	%1 & 0FFFFh				; 段基址1
	db	(%1 >> 16) & 0FFh			; 段基址2，1个字节，8位
	dw	((%2 >> 8) & 0F00h) | (%3 & 0F0FFh)	; 属性1 + 段界限2 + 属性2
	db	(%1 >> 24) & 0FFh			; 段基址3
%endmacro ; 共 8 字节
```

![image-20201222184253858](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201222184253858.png)

怎么实现图中的效果？答案是：

```html
  0F00h
| F0FFH
------------------
  FFFFH
```

这样设置段描述符的属性，挺奇怪的。假如要设置段描述符，应该是这样的：

| 15   | 14   | 13   | 12   | 11   | 10   | 9    | 8    | 7    | 6    | 5    | 4    | 3    | 2    | 1    | 0    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 有效 | 有效 | 有效 | 有效 | 无效 | 无效 | 无效 | 无效 | 有效 | 有效 | 有效 | 有效 | 有效 | 有效 | 有效 | 有效 |

“无效”位置用来设置Limit。

段界限，有20位。段基址，有32位。

一个段描述符，有64位，其中，A、B、C这几块合起来表示段基址，D、E、F这几块合起来表示段界限，G、H、I这几块合起来表示属性。

段基址和段界限，应该理解为一个数值，是某个内存地址的起始地址和结束地址。属性，不是一个数值，而是一个每个bit通过0或1来表示特定含义的属性集合，只不过，为了书写方便，把属性集合写成这些bit集合所表示的数值而已。

同样的段属性集合，属性值可以不同，这是由穿插在其中的无效位（界限值）决定的。（待验证）。

这段汇编代码宏，只设置了三个参数，这不是固定的。只需要，满足段描述符的格式要求就可以了。