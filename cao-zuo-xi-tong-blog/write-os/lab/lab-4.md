# 从低特权级向高特权级转移

## 题外话

本想对照书来规划要做的实验。

否决了这个想法。

理由是，回顾，是比重看更好的学习方法。

先通过回顾来规划实验、做实验。实在回忆不出来，再重复看。

一定要给文章拟定小标题，方便以后阅读时快速定位想看的内容。小标题就是索引。

## 理论

### 特权级转移究竟是什么

特权级转移，是指CPL发生变化吗？

是cs、ip变化后，CPL发生变化了吗？

是访问了与CPL具有不同DPL的数据段吗？

### 场景

用户进程使用系统调用。

### 指令

#### call

不知道怎么写。

call后面接一个函数，这种形式很常见。

这个函数的特权级需要与当前cpl不同，这怎么做到？总不能为一个函数特意创建一个描述符合选择子。

#### jmp

没见过这种用法。

### 门

#### 使用门的规则

1. 调用者的cpl记作CPL_A，调用者的rpl记作RPL_A，被调用者的DPL记作DPL_B，门选择子的RPL不参与检查，门描述符的DPL记作DPL_G。
   1. 门，也有一个选择子。这个选择子指向门描述符。门描述符包含段描述符的选择子还是描述符？
   2. 在特权级检查时，参与检查的是门选择子的RPL还是门描述符的DPL？还是其他？
2. 我只记得这个规则是：调用者的特权级不低于门的特权级 & 调用者的特权级不高于目标的特权级。
   1. 参与的特权级是DPL是CPL？不清楚。
3. 规则，根据分类不同
   1. 一致性代码段
      1. jmp，CPL_A <= DPL_G && CPL_A >= DPL_B && RPL_A <= DPL_G
      2. call，同上
   2. 非一致性代码段
      1. jmp, CPL_A <= DPL_G && CPL_A = DPL_B && RPL_A <= DPL_G
      2. call，CPL_A <= DPL_G && CPL_A <= DPL_B && RPL_A <= DPL_G
4. 概念：
   1. CPL，cs中的DPL。是谁要访问资源，要访问资源的那个访问者的身份。
   2. RPL，目标选择子的RPL。理解不了为何需要RPL。
   3. DPL，目标描述符的DPL。规定访问目标资源的门槛。

#### 门的结构

门的构成：

1. 选择子
2. 偏移量
3. 属性
4. paramCount

选择子 + 偏移量，能确定任意要执行的指令。

属性，存储门的DPL。

paramCount，参数个数。可能不准确。门又不是函数，哪里来的参数。

门也是64个bit，和全局描述符一样。

#### 创建门的宏

![image-20210328091842205](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210328091842205.png)

```assembly
; 四个参数，分别是：偏移量、选择子、ParamCount、属性
%macro
		dword	%1
		dword	%2
		;0001 1111
		;1111 1111 0000 0000 
		;FF00
		dword (%3 & 1Fh) | ((%4 << 8) & FF00h)
		dword	%1 >> 16
%endmacro
```

最难的一句：`dword (%3 & 1F) | ((%4 << 8) & FF00)`。

每次只练习最难的那句。

## 写代码

### 我的方案

#### 系统调用

不是真正的系统调用。

第一次用`retf`转移到特权级2后，在特权级2转移到特权级0，代码段特权级的转移。

### 数据段

在特权级2使用0特权级的数据。

不知道怎么实现。

### 最终方案

我的方案，是我在已有知识的基础上制定的，可能不全面、不正确。

特权级这部分内容，我已经看过两次。现在来实现它，仍然不知道怎么写。能预测，后面其他知识点的实验，会困难重重。

两个实验。

第一个实验，使用门，但是没有特权级转移。

第二个使用，使用门，实现有特权级变化的转移。

#### 使用门--无特权级转移

代码在：`/home/cg/os/pegasus-os/v10`。

很简单。没有压栈，直接使用call调用门描述符的选择子和偏移量，伪代码是：`call 门描述符选择子:偏移量`。

对选择子的使用，就是`选择子：偏移量`。在使用调用门时，`偏移量`是0。

唯一的难点，是门描述符的宏。

最消耗时间点的错误是，`call SelectGate:0`，出现下面的错误：

```shell
 prefetch: EIP [00000001] > CS.limit [00000000]
00015074875e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x0d)
00015074875e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x08)
00015074875i[CPU0  ] CPU is in protected mode (active)
00015074875i[CPU0  ] CS.mode = 16 bit
00015074875i[CPU0  ] SS.mode = 32 bit
00015074875i[CPU0  ] EFER   = 0x00000000
00015074875i[CPU0  ] | EAX=60000043  EBX=00000600  ECX=00090003  EDX=00000012
00015074875i[CPU0  ] | ESP=0000ffc6  EBP=00000000  ESI=000e007c  EDI=0000007a
00015074875i[CPU0  ] | IOPL=0 id vip vif ac vm RF nt of df if tf sf zf af PF cf
00015074875i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
00015074875i[CPU0  ] |  CS:0050( 000a| 0|  0) 00000048 00000000 0 0
00015074875i[CPU0  ] |  DS:0038( 0007| 0|  0) 00000000 ffffffff 1 1
00015074875i[CPU0  ] |  SS:0038( 0007| 0|  0) 00000000 ffffffff 1 1
00015074875i[CPU0  ] |  ES:0038( 0007| 0|  0) 00000000 ffffffff 1 1
00015074875i[CPU0  ] |  FS:0038( 0007| 0|  0) 00000000 ffffffff 1 1
00015074875i[CPU0  ] |  GS:0043( 0008| 0|  3) 000b8000 0000ffff 0 0
```

执行`call`语句时，把call后面的选择子加载到了cs中。正确流程应该是，把选择子中的描述符选择子加载到cs中。

为啥会把门选择子当成了段选择子处理？因为，设置门描述符的属性时，把`s`设置成了1（0表示门等；1表示段描述符）。我复制段描述符进行修改，没有意识到应该修改`s`位。

这是最耗时的一个错误。

我怎么解决的？

1. 知道了错误信息，但不知道为啥会出现这样的错误信息。我分析了门选择子、门描述符、门选择子指向的段描述符和段描述符指向的代码，都没有问题。
2. 反复对比门描述符的结构、于上神的代码和我的代码；短时间用dosbox调试于上神的代码（dosbox不能断点，不实用）。
3. 没有明确目的地反复调试。耗费时间最多。
4. 把门选择子换成它指向的段选择子。一分钟。
5. 重看门结构，找找灵感。三四分钟。

弄错了门描述符的属性，导致耗费了最多时间。

![image-20210328181706186](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210328181706186.png)

这张图说明了每种描述符对应的属性中的type字段，不知道是规定还是有规律。

我把`s`改成了0，仍然不行。还需要修改TYPE。看了于上神的代码，才修改正确。

####  使用门---有特权级转移

##### 回忆

多回忆，少重看，回忆不起来时才重看。

使用门---无特权级转移，流程是：

1. 创建门的目标描述符和描述符指向的代码段
2. 创建门选择子和门的目标描述符的选择子
3. call 门选择子:0。

难点和最大的时间消耗点是门描述符的属性，`s`位是0，type是`C`。

特权级也比较麻烦，试了很多次，才稍微弄清楚一点特权级转移的规则。

使用门---有特权级转移的流程，是怎样的呢？

在使用门---无特权级转移的流程上修改特权级就可以了吗？

根据残存的记忆，不是这样的。

程序初始运行在0特权级，要从低特权级转移到高特权级转移，需要先从0特权级转移到低特权级。我把低特权级设置为3特权级（其他特权级也可以，按照规矩，设置为3特权级）。使用门---有特权级转移的流程是这样的：

1. 从0特权级转移到3特权级。
2. 从3特权级转移到0特权级。
   1. 使用门---有特权级转移来完成。

从代码段转移的视角来看使用门---有特权级转移的流程，是这样的：

1. 实模式、16位代码段
2. 进入保护模式
3. 32位代码段
4. 上面都是0特权级
5. 使用retf进入3特权级大的32位代码段
6. 使用门---有特权级转移切换到0特权级32位代码段
   1. 此处的32位代码段，可以是第3步中的32位代码段，也可以是新建的32位代码段（特权级是0）
   2. 我选择前者。创建门描述符时，偏移量不为0。
   3. 偏移量指向第3步中大的32位代码段的非起点位置。
      1. 遇到一个模糊点。一个代码段中有多个与起点标号平级的标号。
      2. 每个标号的偏移量是物理地址，而不是只使用标号就行。
      3. 无法通过推理确定究竟是怎么回事，只能写代码验证了。
      4. 标号是相对于整个文件的偏移量还是相对于自身所在段的偏移量？标号是相对于ds所在的偏移量。

###### TSS

特权级转移，需要使用TSS。

TSS，任务状态段，类似GDT。这样理解它：

1. 存储一些特定结构的数据。GDT中的每项是一个描述符，占用8个字节。TSS包含很多项。
2. 使用ltr加载到寄存器中。GDT使用lgdt加载到寄存器中。
3. 加载到进来，开发者就不用管了。CPU会自动使用它们。

TSS能保存所有寄存器的值，还保存了3个特权级的堆栈（ss和esp），还有I/O位图、返回位。

但我只使用TSS中的3个特权级堆栈，分别是0、1、2。没有3特权级的堆栈。

TSS中为什么没有3特权级的堆栈？我以前弄明白了这个问题，又忘记了。想再弄清楚，没有清晰思路。先尝试自己理解。

用一个“从高向低转移”的特例来理解。在系统调用函数中使用`reft`，会从当前栈S中出栈低特权级的堆栈（ss和esp）。当前堆栈中的低特权级的堆栈，又是从哪里来的呢？在调用函数时，更具体一些，执行`call`指令时，会把低特权级的堆栈入栈到S中，然后复制函数参数，再入栈cs和eip。

从低到高的转移过程。高特权级的堆栈存储在哪里？在TSS中。换句话说，TSS的用途是为“特权级从低向高转移”时提供目标特权级对应的堆栈。没有比3特权级更低的特权级，也就是说，在特权级从低向高转移时，不会出现比3还低的特权级向3特权级转移。TSS中因此不必存储3特权级的堆栈。TSS中存储的堆栈，是其他更低特权级向高特权级转移时的目标特权特权级的堆栈。这些目标特权级至少有一个比它小的特权级。只有3特权级没有比它小的特权级。

<!--很烦恼。我所写的文章，重看时，需要再次推理。-->

###### I/O位图

是什么？有什么用？

一些伪指令：

1. `db`：一个字节。
2. `dw`：一个字，即2个字节。
3. `dd`：双字，即2个字、4个字节。

TSS的构成：

1. 上一个任务的链接。
2. 3对ss和esp。
3. 通用寄存器。
4. 段寄存器。
5. I/O位图基址。



*于上神的书没有细讲I/O位图，我转去看郑刚的《操作系统真相还原》。这本书的PDF版本目录不全，于是我在微信读书、在搜索引擎找书，想找本目录完整的书。应该花了快半个小时，没有找到。若一开始就直接在现有的PDF中找，已经看明白TSS的I/O位图了。白白浪费了20分钟。教训是，非必要，不要去找书。*



I/O位图的详细讲解在《操作系统真相还原》的第11章（11.1）和第5章（5.4.2）。

![image-20210401143442704](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210401143442704.png)





![image-20210401143901516](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210401143901516.png)



![image-20210401144159974](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210401144159974.png)



TSS的左上角的1个字存储的是`I/O位图在TSS中的偏移量`。在nasm中，偏移量的计算方法是：`$ - 表示TSS的标号`。

那么，应该从这个字开始存储`I/O位图`吗？不是。这个字存储的`I/O位图在TSS中的偏移量`。也就是说，从紧挨着这个字的内存开始存储`I/O位图`。所以，`I/O位图在TSS中的偏移量` = `$ - 表示TSS的标号 + 2个字节`。

最后一个字节是`0xff`。

通用寄存器和段寄存器，每个占用32个bit。没记住顺序不要紧，可以照着书上的结构图写代码。因为不是应试，可以不记住。

##### 最终方案

代码在：`/home/cg/os/pegasus-os/v11`。

1. 进入保护模式。
2. 使用`retf`进入特权级是3的32位代码段ring3。
3. 在ring3使用调用门回到另外一个特权级是0的32位代码段。

写代码时，应该是这样的：

1. 建立3特权级的代码段ring3。

   1. 描述符。

      属性

      | A    | R    | C    | X    | S    | DPL  | DPL  | P    | AVL  | L    | D/B  | G    |
      | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
      | 0    | 0    | 0    | 1    | 1    | 1    | 1    | 1    | 0    | 0    | 1    | 1    |

      把属性从右往左排列。

      `1100 1111 1000`，转换成十六进制，`cf8`。

   2. 选择子。`SelectFlatX_3`。

   3. 代码。标号`LABEL_SEG_PRI3`。

   4. 手工设置描述符的段基址

2. 建立0特权级的代码段ring0。

   1. 描述符
   2. 选择子。`SELECTOR_GDT_GATE`。
   3. 代码。标号：`LABLE_GDT_GATE`。
   4. 手工设置描述符的段基址

3. 建立0特权级的栈

   1. 描述符。`LABLE_GDT_STACK_0`。
   2. 选择子。`SelectStack`。
   3. 代码。标号：`LABLE_STACK`。
   4. 手工设置描述符的段基址

4. 建立调用门

   1. 描述符，一个参数是ring0的选择子。`LABLE_GATE`。dpl是3。
   2. 选择子。`SelectGate`。rpl是3。

5. 建立TSS

   1. 描述符。LABEL_GDT_TSS。最大难点：属性。

      属性。

      | A    | R    | C    | X    | S    | DPL  | DPL  | P    | AVL  | L    | D/B  | G    |
      | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
      | 1    | 0    | 0    | 1    | 0    | 0    | 0    | 1    | 0    | 0    | 0    | 1    |

      把属性从右往左排列。

      `1000 1000 1001`，转换成十六进制，`889`。

   2. 选择子。SelectTss。

   3. 代码：标号`LABEL_TSS`。

      1. 0、1、2三个特权级的堆栈(ss和esp)
      2. I/O位图在TSS中的偏移量
      3. I/O位图结束符：`0xff`。
      4. 通用寄存器和段寄存器。

   4. 手工设置描述符的段基址

6. 加载TSS

   1. 指令`ltr TSS选择子`

7. 在ring3中使用调用门

## 调试

``&' operator may only be applied to scalar values`

`bx_dbg_read_pmode_descriptor: selector 0x0060 points to a system descriptor and is not supported!
(0) [0x000000020541] 001a:0000000000000014 (unk. ctxt): callf 0x0060:00000000     ; 9a000000006000`



```shell
bx_dbg_read_pmode_descriptor: selector 0x0052 points to a system descriptor and is not supported!
(0) [0x000000020511] 0012:0000000000000011 (unk. ctxt): callf 0x0052:00000000     ; 9a000000005200
<bochs:26> s
00030149769e[CPU0  ] fetch_raw_descriptor: GDT: index (ff57) 1fea > limit (57)
00030149769e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x0a)
00030149769e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x08)
00030149769i[CPU0  ] CPU is in protected mode (active)
00030149769i[CPU0  ] CS.mode = 32 bit
00030149769i[CPU0  ] SS.mode = 32 bit
00030149769i[CPU0  ] EFER   = 0x00000000
00030149769i[CPU0  ] | EAX=60000a33  EBX=00000600  ECX=00090002  EDX=00000012
00030149769i[CPU0  ] | ESP=000001ff  EBP=00000000  ESI=000e007c  EDI=0000007a
00030149769i[CPU0  ] | IOPL=0 id vip vif ac vm RF nt of df if tf sf zf af PF cf
00030149769i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
00030149769i[CPU0  ] |  CS:0012( 0002| 0|  2) 00020500 ffffffff 1 1
00030149769i[CPU0  ] |  DS:0000( 0007| 0|  0) 00000000 ffffffff 1 1
00030149769i[CPU0  ] |  SS:001a( 0003| 0|  2) 00020540 001fffff 1 1
00030149769i[CPU0  ] |  ES:0000( 0007| 0|  0) 00000000 ffffffff 1 1
00030149769i[CPU0  ] |  FS:0000( 0007| 0|  0) 00000000 ffffffff 1 1
00030149769i[CPU0  ] |  GS:0043( 0008| 0|  3) 000b8000 0000ffff 0 0
```



## 最大的问题

做"使用调用门从低特权级转移到高特权级的时候"，遇到的最大难题是：

```shell
bx_dbg_read_pmode_descriptor: selector 0x0052 points to a system descriptor and is not supported!
(0) [0x000000020511] 0012:0000000000000011 (unk. ctxt): callf 0x0052:00000000     ; 9a000000005200
<bochs:26> s
00030149769e[CPU0  ] fetch_raw_descriptor: GDT: index (ff57) 1fea > limit (57)
00030149769e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x0a)
00030149769e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x08)
00030149769i[CPU0  ] CPU is in protected mode (active)
00030149769i[CPU0  ] CS.mode = 32 bit
00030149769i[CPU0  ] SS.mode = 32 bit
00030149769i[CPU0  ] EFER   = 0x00000000
00030149769i[CPU0  ] | EAX=60000a33  EBX=00000600  ECX=00090002  EDX=00000012
00030149769i[CPU0  ] | ESP=000001ff  EBP=00000000  ESI=000e007c  EDI=0000007a
00030149769i[CPU0  ] | IOPL=0 id vip vif ac vm RF nt of df if tf sf zf af PF cf
00030149769i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
00030149769i[CPU0  ] |  CS:0012( 0002| 0|  2) 00020500 ffffffff 1 1
00030149769i[CPU0  ] |  DS:0000( 0007| 0|  0) 00000000 ffffffff 1 1
00030149769i[CPU0  ] |  SS:001a( 0003| 0|  2) 00020540 001fffff 1 1
00030149769i[CPU0  ] |  ES:0000( 0007| 0|  0) 00000000 ffffffff 1 1
00030149769i[CPU0  ] |  FS:0000( 0007| 0|  0) 00000000 ffffffff 1 1
00030149769i[CPU0  ] |  GS:0043( 0008| 0|  3) 000b8000 0000ffff 0 0
```

`bx_dbg_read_pmode_descriptor: selector 0x0052 points to a system descriptor and is not supported!`这只是“警告”，不需要理会，不影响代码执行结果。可是，我一开始认为这是错误。怎么知道它不是错误？我运行了“调用门---无特权级转移”的代码，也有这个信息，但是代码却能够正常执行。

`fetch_raw_descriptor: GDT: index (ff57) 1fea > limit (57)`这个才是错误信息。

不得不说，bochs的错误信息太鸡肋了！其他语言，比如PHP，数组越界了，执行时会明白地告知开发者数组越界了。可开发者从上面的信息，不能获得任何线索。**我就在这个问题上耗费了10多个小时**。这10个小时，我做了些什么？不记得了。因为，我没有明确的思路，只有一个模糊的想法，而且，想法太多了。我一直运行啊运行啊，最终彻底失望。如果我也这样拆炸弹，我已经被炸死很多次了。

可是，毫无头绪，我能怎么办？我多次告诫自己：先想出一个思路，然后通过运行去验证它。

为了解决问题，我尝试了下面这些方法：

1. 调整代码，例如，修改门选择子的特权级、门描述符的特权级等。
2. 去掉手工设置描述符中的段基址中的物理地址。
3. 单步执行，看看究竟是哪一步导致错误。
4. 对比于上神的代码。
5. 把于上神的代码中的TSS复制过来。

在10多个小时中，我大概一直在做上面这些事情。请记住一个事实：调试程序，非法非常非常耗费时间。

今天早上，我再次碰碰运气，执行第5个步骤，成功解决了问题。

昨天，用同样的方法，我为啥没有解决问题？思路混乱，可能复制的时候出错了。

在vim中复制粘贴代码，我总是感觉非常不顺手。可github经常出现网络问题，我不能方便地在虚拟机和物理机之间同步代码。

思路不清晰，就用笔算。记住，越不清晰，越要用笔算，不要用心算。



```assembly
READ_FILE_OVER:
        ;mov al, 'O'
        ;mov ah, 0Dh
        ;mov [gs:(80 * 23 + 33) * 2], ax
        ; 开启保护模式 start
        ;cli
        ;mov dx, BaseOfLoaderPhyAddr + 0
        xchg    bx, bx
        mov     dword [GdtPtr + 2], BaseOfLoaderPhyAddr + LABEL_GDT
        lgdt [GdtPtr]

        ; 加载TSS
        mov ax, SelectTss
        ltr ax

        cli
```

上面的代码，执行时，出现下面的错误：

```shell
LTR: not recognized in real or virtual-8086 mode
```

在保护模式下才能加载 tss。



### LDT

![image-20210401164839050](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210401164839050.png)

### TSS描述符

![image-20210401164941653](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210401164941653.png)

### 疑问

TSS中的I/O位图的偏移地址是：`$ - TSS的标量 + 2`。`2`的单位是字节。为什么不是字？其他元素比如通用寄存器，都把“保留位”计入了元素占用的空间，占用空间是32个bit。可I/O位图的偏移地址为什么只占用16个bit而不是32个bit？



## 其他

电路仿真软件



下载地址

https://sourceforge.net/projects/circuit/

教程

http://www.cburch.com/logisim/download.html

https://vlab.ustc.edu.cn/guide/index.html



## 电子书资源站

https://www.52doc.com/category/6

https://www.manongbook.com/

