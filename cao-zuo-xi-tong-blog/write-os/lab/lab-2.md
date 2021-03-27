# 从保护模式切换到实模式

之前耗费了非常多时间才弄明白的问题，如今只残存着模糊的记忆。

很神奇，刚写完文章理清写代码的思路，我居然不能快速想起我的思路是什么。

不纠结这个了。我把事情分拆成两步：

1. 想清楚，代码怎么写。
2. 写代码。

写代码时，直接对照第一步的思路写，不要再重新构思。若重新构思，干嘛做第一步？

写代码，很容易变得不清醒，有了第一步的构思，不清醒的时候，看一眼就清醒了。

## 理论

从保护模式切换到实模式，流程如下：

1. 从32位模式进入16位模式，仍然是保护模式。但是，修改了cs，cs是16位模式，即描述符的属性是16位而不是32位的。
   1. 描述符的属性是难点。
2. 从16位模式跳转到16位模式
   1. 先切换到实模式
   2. 再跳转
      1. 实现跳转的方式很特别。通过直接修改二进制代码实现。
   3. 不确定二者的顺序，只能通过测试。

从保护模式切换到实模式，必须经过一个16位模式（保护模式）的中间环节。为了弄清楚”为什么“，我曾经耗费了非常多时间，现在仍记得一点，暂时不写出来。

从16位模式（保护模式）跳转到16位模式（实模式）的方法：

1. 在16位模式（保护模式）设置一个标量，伪代码如下：

   ```c
   GO_BACK_REAL_MODEL:
   		jmp 0: IN_REAL_MODEL
   ```

2. 在16位模式（实模式）修改标量`GO_BACK_REAL_MODEL`表示的二进制代码，伪代码如下：

   ```c
   // 修改标量`GO_BACK_REAL_MODEL`表示的二进制代码
   .IN_REAL_MODEL:
   			// 回到实模式了
   ```

两个疑点：

1. `GO_BACK_REAL_MODEL`应该是`GO_BACK_REAL_MODEL`还是应该是`.GO_BACK_REAL_MODEL`？即，前面需要点号吗？
   1. 不加点号，是函数？不纠结这点，到时候在代码中测试。
2. 怎么修改标量`GO_BACK_REAL_MODEL`表示的二进制代码？
   1. 和二进制代码的格式有关，需要看书中的说明。
   2. 不看书，自己查看二进制代码。可是，没有头绪。
      1. 写一个汇编文件，只包含语句`jmp 0: IN_REAL_MODEL`，编译成二进制文件然后查看。

## 实验

代码在：

1. `/home/cg/os/pegasus-os/v7`
2. `/home/cg/os/pegasus-os/v8`

### 16位模式代码（保护模式）

需要完成这些事情，小标题就是需要完成的事情。

写得挺难受的，做下面每件事的方法都模糊不清。

#### 设置除cs位的段寄存器

这些段寄存器是`ds、es、ss、fs、gs`。

把这些段寄存器设置成什么值？

目前，仍然是保护模式，段寄存器的值只能是选择子。这个选择子必须符合实模式下的值的要求。有什么要求？

1. 描述符属性必须是16位。
2. 段基址是0。
3. 段界限不能超过2的16次方。

1和3是最重要的。我不知道上述说明是否正确，写代码测试吧。

具体做法是：

1. 新增描述符。
2. 新增描述符对应的选择子。

16位模式代码段（保护模式）的描述符`LABLE_GDT_FLAT_WR_16`的编写过程如下。对应的选择子是`SelectFlatWR_16`。

`LABLE_GDT_FLAT_WR_16`的属性：

可读写段：

| A    | W    | E    | X    | S    | DPL  | DPL  | P    | AVL  | L    | D/B  | G    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 0    | 1    | 0    | 0    | 1    | 0    | 0    | 1    | 0    | 0    | 0    | 1    |

`D/B`位是0，表示16位指令。`G`如何设置？不知道。我们这个描述符，应该不关注吧。

那么，属性值是：`1000 1001 0010`，转换成十六进制是，`0892h`。

| A    | R    | C    | X    | S    | DPL  | DPL  | P    | AVL  | L    | D/B  | G    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 0    | 0    | 0    | 1    | 1    | 0    | 0    | 1    | 0    | 0    | 0    | 1    |

属性值是：`1000 1001 1000`，转换成十六进制是，`0898h`。





##### 手工设置描述符的段基址

![image-20210326115438498](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210326115438498.png)

怎么做？

1. 把描述符的标号记作`D`，16位模式代码（保护模式）的标号是`F`。

2. 设置`D`的第16位~~~第39位是?

   1. 这个运算，又是难点。不知道怎么做。
   2. 看了于上神的代码，费了两三分钟，理解了。计算方法是：
      1. 计算出标号`F`的内存地址：cs * 16 + F。把结果记作`NF`。
      2. 把`NF`拆分成三部分：
         1. 第0~~~第15位
         2. 第16~~~第23位
         3. 第24~~~第31位
      3. 依次复制到`D`的三部分：
         1. 第16~~第31位
         2. 第32~~第39位
         3. 第56~~~第63位

   代码是：

   ```assembly
   xor		eax,	eax
   mov		eax,		cs
   shl		eax,		4
   add		eax,		LABEL_SEG_16
   
   mov		word [LABLE_GDT_FLAT_X_16+2],	ax
   shr		eax,		16
   mov		byte [LABLE_GDT_FLAT_X_16+4],  al
   mov		byte [LABLE_GDT_FLAT_X_16+7],	ah
   ```

   

#### 切换到实模式

无难度，固定流程。

1. 关闭A20。
2. 关闭cr0的pe位。

与切换到保护模式时对上面两个要素的操作反着来。

我不记得具体代码，可看之前的代码。没必要刻意熟记每个API。

#### 跳转到实模式的代码

```assembly
GO_BACK_REAL_MODEL:
		jmp 0:IN_REAL_MODEL
```

标号`GO_BACK_REAL_MODEL`的作用是在16位模式代码（实模式）中修改二进制代码。

### 16位模式代码（实模式）

#### 修改二进制代码

```assembly
mov		[GO_BACK_REAL_MODEL], 


.IN_REAL_MODEL:
		; 返回16位模式代码（实模式）后所做的事情
```

怎么写这段代码，完全没有头绪，有许多问号。

1. 修改标号的哪几个bit？好解决。

2. 标号名称应该加点号吗？通过测试解决。

3. 伪代码是这样的吗？通过测试解决。

   ```c
   mov		ax,   [GO_BACK_REAL_MODEL]
   // 位移，获取ax中需要调整的bit位，假设结果是n-ax
   // 把n-ax中的相应bit调整为 IN_REAL_MODEL
   mov		n-ax, [IN_REAL_MODEL]  
   ```

#### 返回16位模式代码（实模式）

需要做什么？非常模糊。我现在所写的内容，是我理解了、残存在记忆中的别人的做法。

打印一个字符，让调试者也就是我看到CPU已经返回了实模式。

这种测试方法不严谨，更好的方法是，在这里使用大于5M的内存，若报错，就说明已经回到实模式了。暂时搁置。

上面这件事，简单。

很困难的，是这件事。

1. 关闭cr0的pe位，是在这里做吗？
2. 需要更新段寄存器吗？

从理论上，我回答不了，只能全部依靠测试。

尝试从理论上推理一下。

1. 在16位保护模式下切换到实模式，cs寄存器的值符合要求吗？
   1. 对cs的值的要求是什么？完全不知道。
      1. 16位代码。从32位代码进入本代码时设置。具体说，是选择子指向的描述符设置的。
      2. 读写属性。可读可写？还是只读？不知道。
   2. 切换到了实模式，跳转回16位代码模式（实模式）的语句`jmp 0:IN_REAL_MODEL`正好可以运行啊。
      1. 这个结论，支持在本段切换到实模式。
2. 再从理论上推理已经没有意义了。断章取义的推理，在知识不全时看起来很有道理。留着写代码测试吧。

### 32位模式代码（保护模式）

跳转到16位模式代码（保护模式）。



```shell
<bochs:9> xp /1wx 0x20000:0x00020438
[bochs]:
0x0000000000020438 <bogus+       0>:	0xfeebfeeb
<bochs:10> c


xp /1wx 0x0010:0x0000

<bochs:21> xp /1wx 0x000000000002013e
[bochs]:
0x000000000002013e <bogus+       0>:	0x00000000

  xp /1wx 0x0000000000020146
  

xp /1wx 0x20000:0x00020430

<bochs:46> xp /1wx 0x20000:0x00020430
[bochs]:
0x0000000000020430 <bogus+       0>:	0xfeebfeeb

# 使用内存地址访问数据。进入保护模式后，再访问试试。仍然能正确访问到数据。
# 使用选择子，xp /1wx 0x0010:0x0000，无法访问到正确的数据。
# 原因，手工设置描述符的段基址失败了吗？
# 怎么验证是否失败了？
xp /1wx 0x0000000000020430

<bochs:46> xp /1wx 0x20000:0x00020430
[bochs]:
0x0000000000020430 <bogus+       0>:	0xfeebfeeb
<bochs:47> xp /1wx 0x0000000000020430
[bochs]:
0x0000000000020430 <bogus+       0>:	0xfeebfeeb
<bochs:48> 


```



手工设置 段基址成功。但是，进入保护模式后，不能使用选择子访问到那个地方的数据。



`0450f000`

`100010100001111000000000000`

`100       0101 0000      1111 0000     0000 0000`

`00000000 00000000 00000000 00000000 00000100 01010000 11110000 00000000`

```shell
<bochs:12> r
CPU0:
rax: 00000000_00020450
rbx: 00000000_00000500
rcx: 00000000_00090001
```







0028--->10 1    000   ---> 101---->5

0033--->11 0011 ---> 110 011 ---> 110---> 6

````shell
<bochs:20> sreg
es:0x0028, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
cs:0x0008, dh=0x00cf9b00, dl=0x0000ffff, valid=1
	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
ss:0x0028, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
ds:0x0028, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
fs:0x0028, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
gs:0x0033, dh=0x0000f30b, dl=0x8000ffff, valid=7
	Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0000, dh=0x00008200, dl=0x0000ffff, valid=1
tr:0x0000, dh=0x00008b00, dl=0x0000ffff, valid=1
gdtr:base=0x000000000002013e, limit=0x37
idtr:base=0x0000000000000000, limit=0x3ff
<bochs:21> xp /1wx 0x000000000002013e
[bochs]:
0x000000000002013e <bogus+       0>:	0x00000000
<bochs:22> xp /1wx 0x0000000000020146
[bochs]:
0x0000000000020146 <bogus+       0>:	0x0000ffff
````

`es:0x0028`中的`0x0028`是描述符的选择子。

和下面的代码是吻合的。

```assembly
LABEL_GDT:      Descriptor  0,  0,      0
LABLE_GDT_FLAT_X: Descriptor    0,              0ffffffh,                0c9ah
LABLE_GDT_FLAT_X_16: Descriptor 0,              0ffffffh,                98h
;LABLE_GDT_FLAT_X: Descriptor   0,              0FFFFFh,                 0c9ah
;LABLE_GDT_FLAT_WR:Descriptor   0,              0fffffh,                 293h
LABLE_GDT_FLAT_WR_TEST:Descriptor 5242880,              0fffffh,                 0c92h
LABLE_GDT_FLAT_WR_16:Descriptor 0,              0fffffh,                 0892h
LABLE_GDT_FLAT_WR:Descriptor    0,              0fffffh,                 0c92h
LABLE_GDT_VIDEO: Descriptor     0b8000h,                0ffffh,          0f2h

GdtLen  equ             $ - LABEL_GDT
GdtPtr  dw      GdtLen - 1
dd      BaseOfLoaderPhyAddr + LABEL_GDT
SelectFlatX     equ     LABLE_GDT_FLAT_X - LABEL_GDT
SelectFlatX_16  equ     LABLE_GDT_FLAT_X_16 - LABEL_GDT
SelectFlatWR    equ     LABLE_GDT_FLAT_WR - LABEL_GDT	
SelectFlatWR_TEST       equ     LABLE_GDT_FLAT_WR_TEST - LABEL_GDT
SelectFlatWR_16 equ     LABLE_GDT_FLAT_WR_16 - LABEL_GDT
SelectVideo     equ     LABLE_GDT_VIDEO - LABEL_GDT + 3
```

## 总结

大概耗费了17个小时。前15个小时，毫无进展，令人沮丧。后面无意中改正确了，进展非常快，势如破竹。

为什么进展非常慢？

这些汇编代码是我20多天前写的，这次重新加新功能，看不懂，写好了代码，有问题，不知道错在哪里。

时间，消耗在漫无目的的无脑、重复运行代码上。

后来，知道怎么验证代码是否正确。

例如`jmp selector:0`，执行跳转语句就报错。我后来知道如何查看`selector:0`是否指向正确的机器码。

发现跳转目标指向错误的数据后，我再检查手工设置全局描述符的段基址是否正确。

看段基址是否正确，再看这个段基址是否正确地存储到了描述符的相应的bit位。

通过GDT BASE查看描述符来查看段基址是否存储到了描述符的相应的bit位。

在实模式中，标量是相对于段基址的偏移量。可以这样理解，在一个文件中，标量是相对这个文件开头位置的偏移量。内存中的数据，只不过把文件开头换成了这个段的基址。

只根据偏移量是不能获得标号所指示的数据的。需要使用内存地址（绝对内存地址）才能获得标号所指示的数据。

实模式下，`cs`的值有时会与loader在内存中的物理地址相同。

实模式下，`mov [variable], ax`实质是`mov [ds:variable],ax`。忽略了`ds`，可能是我一直遇到错误的原因。

使用描述符能指代任意代码段。

调试的时候，要善于使用`jmp $`这样容易识别的机器码来查看二进制代码。例如，我想知道`jmp selector:0`是不是指向了正确的数据，只需在目标数据的位置填充`jmp $`。能看到`jmp $`，说明它指向了正确的数据。

想知道汇编的机器码，只需写一段汇编代码，然后编译，使用xxd查看就行。就像下面这样的：

```assembly
mov     ax,     cs
mov     ds,     ax
```

上面的汇编代码不需要讲究格式、上下文，几乎所有合法的汇编指令都行，只有编译不报错。

然后编译，`nasm -o t3 t3.asm`。

再使用`xxd`查看，`xxd t3`，数据如下：

```shell
[root@localhost v8]# nasm -o t3 t3.asm
[root@localhost v8]# xxd t3
00000000: 8cc8 8ed8                                ....
[root@localhost v8]#
```

`mov  ax,  cs`的机器码是`8cc8`。使用bochs断点查看到的二进制数据和使用`xxd`查看到的数据的顺序不一致。这和”小端法“有关。

