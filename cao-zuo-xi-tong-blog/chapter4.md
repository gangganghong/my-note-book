# 让操作系统走进保护模式

## FAT12

FAT12是一个DOS时代的文件系统。

文件系统，是一种提供在磁盘等存储设备上提供数据读写功能的软件，自身也需要存储在存储设备上。

FAT12的结构是：引导扇区、2个FAT区域、根目录、数据区。

引导扇区也是一个扇区，所以，它的大小是512字节。引导扇区是整个软盘的第0号扇区。

一个FAT结构的大小是9个扇区，2个FAT结构所占空间是18个扇区。2个FAT结构的内容相同，互为对方的冗余、副本。

根目录区位于第二个FAT表之后，初始位置是19号扇区，第20个扇区。根目录区由若干个目录条目区（Directory Entry)组成，条目最多有BPB_RootEntCnt个。由于根目录区的大小是依赖BPB_RootEntCnt的，所以长度不固定。

根目录的每个条目占用32字节。根目录区条目的格式包含下列内容：文件名8字节、扩展名3字节，此条目对应的开始簇号，文件大小，文件属性，最后一次写入时间，最后一次写入日期，保留位。

数据区的第一个簇号是2，不是0或1。若目标文件的开始簇号是2，那么，此文件的数据开始于数据区的第一个簇。

在软盘中查找文件的方法是，遍历根目录，检查每个根目录区的每个条目的文件名是不是目标文件名，若是，这个根目录区条目就是要找的文件的根目录区条目，此根目录区条目中的开始簇号就是目标文件的第一个簇号X。在FAT表中找到第X个条目。

可以用xxd查看存储在FAT12系统中的数据。

引导扇区中的结构成员分为两类：前缀为BS和前缀为BPB。

BPB的全称是BIOS Parameter Block。

数据区的第一个扇区在哪里？需要计算出根目录区占用多少个扇区。假设根目录占用扇区RootDirSectors个，计算公式如下：

RootDirSectors = (BPB_RootEntCnt * 32 + 每个扇区占用的大小-1) / 每个扇区占用的大小。

这么怪异而不直观的公式，原因是，当根目录占用扇区不足一个扇区时，根目录扇区占用的扇区数应该是1；根目录占用扇区大于一个扇区而不足两个扇区时，根目录扇区占用的扇区数应该是2。理解这个公式，都让我费了点功夫。

每个FAT项占用12个字节，包括一个字节和另一个字节的一半。这意味着，一个FAT项可能会跨域两个扇区。

每12位称为一个FAT项，第0个、第1个FAT项从来不使用，从第2个FAT项开始使用。

FAT项的值代表文件的下一个簇号。如果值大于等于0xFF8，表示当前簇是文件的最后一个簇。如果值是0xFF7，表示当前簇是一个坏簇。

一个簇只有一个扇区，所以，簇号即扇区号。FAT项的值表示的是数据区的簇号，计算这个簇号在磁盘中的位置时，需要加上前面的19个扇区。

## DOS可以识别的引导盘

我要开发的操作系统和DOS有什么关系？

什么样的引导盘才能让DOS可以识别？

引导扇区加上BPB头等信息才能让引导盘被DOS识别，同时才能被Linux识别。

BPB头等信息分为两大类，分别是前缀为BPB_和前缀为BS_的数据。

书上没有写清楚为啥有BPB头等信息就可以被DOS和Linux识别。

第一条指令是跳转指令，第二条是让CPU空转的指令，如下：

```assembly
jmp	LABEL_START		; Start
nop
```



BPB头等数据不需要在软盘的最开始位置吗？

## 一个最简单的loader

初始内存地址是 0x100，使用 `orgin 0100h` 完成。

在屏幕上打印字符 L，方法是往显存写入 L，指令是 `mov al 'L'   mov [gs:(80*0+39)*2], al` 。

然后，是一个死循环，指令是 `jmp $` 。

把这个loader编译成 COM 文件，可以在 FreeDos 中直接运行。注意，不要使用 DOSBox，使用很不方便。使用 bochs 中的 FreeDos，很方便。我曾经成功使用过，不过，可能没写文档，也可能不知道把文档放到哪里了。再使用它时，一定要写准确的文档。

## 加载loader入内存

### 从软盘中读取loader

#### 查找loader

软盘中有多个文件，怎么从软盘中读取loader呢？

首先，要找到软盘中loader的数据元信息，即它的第一个扇区的位置信息，所有扇区的位置信息。

这一切，可以从FAT12文件系统的根目录区获取。

根目录区的第一个扇区是软盘的第19个扇区（引导扇区 + 两个FAT占用的18个扇区）。根目录中有很多个根目录项，数量和软盘中的文件数量相同，一个根目录项对应一个文件。每个根目录项占用32个字节，查找文件会使用根目录项中的“文件名”和”第一个簇号“两个成员变量（二进制数据中没有这个概念，这里借用C语言中的struct的成员变量概念）。“文件名”在根目录项的初始位置，即偏移量是0。“第一个簇号”，是指文件在数据区的第一个簇的簇号，在根目录项中的偏移量是 `0x1A`，即26个字节。由于一个簇只有一个扇区，所以，“第一个簇号”，也就是“第一个扇区”。

从根目录中获取文件名后，逐个字符与字符串”LOADER BIN"(这是loader在软盘中的名称)对比，若完全匹配，说明loader在软盘中存在，进入读取loader的流程。若不完全匹配，说明当前根目录条目不是loader的根目录条目，继续读取下一个根目录条目，重复上面的流程。

显然，这是一个遍历根目录的过程。遍历终止的条件是“第一个扇区”等于 `0xFFFF`。若“第一个扇区”是`0xffff'，表示这是文件的最后一个簇，（为何如此？）不必读取`0xffff`这个扇区的数据。

怎么遍历根目录项目？

使用BIOS中断`int 15h`。它的使用方法，类似C语言中的函数，提供参数，返回结果。这个中断的使用流程如下：

1. 重设软驱。
2. 读取数据。

必须严格按照这个先后顺序来执行。两个流程都使用`int 15h`，只不过参数不同。

重设软驱需要的参数是：（忘记了）？

读取数据需要的参数是：柱面号、磁头号、在软盘中的扇区号（需要将参数放入规定的寄存器，忘记了）；返回数据在es:bx指令的内存区域。

读取第一个根目录项的参数是：

1. 柱面号：？
2. 磁头号：？
3. 扇区号：1（引导扇区）+18（两个FAT) = 19。

柱面号和磁头号需要计算，方法是：

1. 扇区号 / 一个磁道包含的扇区数 = X，余数是Y。
2. 柱面号 = X >> 1。
3. 磁头号 = X & 1。
4. 初始扇区号 = Y + 1。

初始扇区号，并不是在软盘中的累加扇区号，而是该扇区在当前柱面、当前磁头的初始扇区号。为啥要加1？因为CHS规则下，A柱面A磁头下的扇区号的初始值是1，所以，最终的扇区号应该是扇区偏移量 + 1。

注意，初始扇区号中的 + 1 又让我困扰了不少时间。

> CHS(0,0,1)（0柱面-0磁头-1扇区）就是硬盘的逻辑（LBA）第0扇区，MBR就储存在这个扇区上。

文件的第一簇的数据，通过它对应的根目录项中的“第一簇”可以读取到。剩余的数据，需要根据“第一簇”来读取FAT项来获取。

“第一簇”既是文件在文件区的第一块数据的簇号，也是它在FAT1中的FAT项的编号，例如，若“第一簇”是2，那么，在FAT中对应的FAT项的编号是2。文件由多少个簇，就有多少个对应的FAT项。簇和FAT项一一对应。

查找文件所有簇号的流程是：

1. 从文件对应的根目录项中获取文件的第一个簇的编号N，在FAT中找到编号为N的FAT项。
2. FAT项的值是文件的下一个簇的簇号，也是这个簇对应的FAT项的编号。它就是一个单链表。第一个簇号是头结点。
3. FAT项的值有三种特殊值：
   1. 值是0XFFFF，表示当前FAT项是最后一个。
   2. 大于某个值，坏簇。
   3. ？（忘记了）

#### GetFATEntry

这个函数，参数是FAT项编号，返回值是FAT项的值。

一个FAT项占用12字节，所以，它可能跨越两个扇区。为了完整获取每个FAT项，每次需要获取包含此FAT项的两个扇区。根据FAT项能计算出这个FAT项的字节偏移量是多少，进而计算出它的扇区偏移量，为读取软盘创造条件。

难点是：检查FAT项的字节偏移量是不是整数个字节。

如果是整数个字节，[es:ax]为起点的16字节的低12位，是结果。

如果不是整数个字节，[es:ax]为起点的16字节的高12位，是结果。

结果是文件的下一个簇在数据区的簇号。注意这个簇号是相对于数据区的簇号，在读取软盘时需转换为A柱面A磁头的扇区号。

### 读取loader

从文件对应的根目录项的“第一个簇”获取文件在数据区的第一簇数据，同时根据“第一簇”找到编号为“第一簇”的FAT项。这个FAT项的值是下一簇数据的簇号。

已知文件的簇号，能计算出这个簇在软盘的扇区偏移量，进一步能计算出这个簇（扇区）在A柱面B磁头的扇区号。

已知FAT项的编号，能计算出这个FAT项在FAT区域的字节偏移量，进而计算出FAT项在FAT区域的扇区偏移量，再进一步计算出FAT项在软盘的扇区偏移量，最终能计算出这个簇（扇区）在A柱面B磁头的扇区号。

已知柱面号、磁头号、扇区号，就能使用BIOS中断`int 15h` 读取软盘数据。

写的时候，仍需要想一想。若写作场景再严格一点（比如面试），我担心自己不能如此从容地写出来。

### 公式

能使用的内存有多少？足够加载loader吗？

实模式下，能使用的最大内存是1M。软盘容量是1.44M。loader最大是1M吗？不是，必须小于1M。那么，loader最大是多少？

loader是怎么到软盘上的？

复制到软盘上的。用几个linux命令复制到软盘上的。

如何在软盘上找到loader？

使用BIOS中断 `int 13h`。它像高级语言中的函数，需按要求提供参数给它。我之前使用过BIOS中 `int 15h` 来获取内存。

还是书本上的讲解能抓住重点。`int 13h` 需要的参数是柱面号、磁头号、当前柱面上的初始扇区号，而不是软盘的0号扇区。

一个软盘有2个磁头，每个盘面有80个磁道，每个磁道有18个扇区，每个扇区有512字节，所以，一个软盘的大小是：
$$
2 * 80 * 18 * 512 = 1440 byte
$$
1.44M，是个估算值。

磁头号、柱面号、起始扇区号是怎么计算出来的？已知条件是扇区号，这个扇区号是在整个软盘中的扇区号。我不明白，这个扇区号是怎么得到的。

在已知整个软盘中的扇区号的前提下，计算步骤如下：

1. 扇区号除以每个磁道包含的扇区的数量，商是Q，余数是Y。

2. 柱面号 = Q >> 1。

   1. 对此，我有疑问。

   2. Q是磁道数，每个柱面有2个磁道，一共有多少个柱面呢？是磁道的一半。

   3. 对这个结果，我本无异议。可是，代入具体数据，却发现计算结果与正确结果有差异。

      1. 当Q = 0时，柱面号是0。
      2. 当Q = 1时，柱面号是0。
      3. 当Q = 2时，柱面号是1。
      4. 当Q = 3时，柱面号是1。
      5. 当 Q = 4时，柱面号是2。
      6. 当 Q = 5 时，柱面号是2。
      7. 当 Q=6时，柱面号是3。

   4. 柱面的初始编号是0还是1？如果是0，上面的例子反倒证明

   5. $$
      柱面号 = Q / 2
      $$

      是正确的。若柱面的初始编号是1，那么，计算柱面号的公司应该修改为：
      $$
      柱面号 = （Q / 2) + 1
      $$

3. 磁头号 = Q & 1。
   1. 又不理解。
   2. 磁头号要么是0，要么是1。用具体数据来理解吧。
      1. Q = 0，0 & 1 = 0。磁头号是0，即扇区在第0盘面。
      2. Q = 1,  1 & 1 = 1。磁头号是1，即扇区在第1盘面。
      3. Q = 2, 2 & 1 = 10b & 1 = 0。
      4. Q = 3, 3 & 1 = 11b & 1 = 1。
   3. 这个公式，计算出来的结果，是正确的。可是，不看这本书，我不可能推导出这个公式。这个公式是怎么想出来的？
   4. 判断磁道的序号（从1开始，第N个磁道）的最低位是否为1就能判断这个磁道在哪个盘面，为什么？
   5. 这和操作系统的任何领域知识无关，只是个应该很容易的数学计算而已。我却花了很多时间不能推导出来。

计算磁头号的方法

https://blog.csdn.net/jj9876jj/article/details/5252329?utm_medium=distribute.pc_relevant.none-task-blog-baidujs_title-2&spm=1001.2101.3001.4242

不必使用这么复杂的方法。

有两个磁头，意味着同一个编号的磁道有两道。扇区号 / 每个磁道包含的扇区的数量 = 该扇区在第多少个磁道 C。第奇数个磁道在1磁头，第偶数个磁道在0磁头（序号初始值是0）。怎么会有这么个结论？归纳法。

1. 第0个磁道，在0磁头。
2. 第1个磁道，在1磁头。
3. 第2个磁道，在0磁头。
4. 第3个磁道，在第1磁头。

只需要判断C是偶数还是奇数就能知道该扇区所在的磁头数。而检查一个数是奇数还是偶数的方法是：C & 1 。它的结果是0，偶数；它的结果是1，奇数。若要问为什么？这是常识。

若有三个磁头，磁道数 = C / 3。余数是磁头号。

这么一个简单的问题，花费了2个多小时。没有明确的思考思路。找到了网上的复杂的做数学题的计算方法，没看懂。但我突然明白了如何理解作者这个计算公式。

起始扇区号怎么计算？

这里的起始扇区号是指在A柱面C磁头的起始扇区号。起始扇区号 = 余数 + 1。为啥要加1？因为各磁道内的扇区编号的起始编号是1。余数是在磁道内的偏移量。当偏移量是0时，扇区号是1；当偏移量是1时，扇区号是2。

### 代码

### int 13h

代码中的难点，是对 `div` 指令的使用。

| 被除数  | 除数     | 商   | 余数 |
| ------- | -------- | ---- | ---- |
| ax      | reg/mm8  | AL   | Ah   |
| Dx:ax   | reg/mm16 | ax   | Dx   |
| Edx:eax | reg/mm32 | eax  | edx  |



`int 13h` 中断，参数如此套路化，我自己写，也只能写出一个一模一样的。调用参数如此多，不花点时间，我哪里能记得住呢？怎么才叫自己写？

使用 `int 13h`，首先要复位软驱，需提供的参数是：ah = 00h，dl=驱动器号（0表示A盘），代码是：

```assembly
xor	ah, ah	; `.
xor	dl, dl	;  |  软驱复位
int	13h	; /
```



注意咯。`int 13h` 因参数不同，具有不同的功能，一个是软驱复位，一个是读取软盘。

读取软盘需要的参数是：

| 寄存器 |                             参数解释 |
| ------ | -----------------------------------: |
| ch     |                               柱面号 |
| cl     | 起始扇区号，在A柱面B磁头的起始扇区号 |
| dh     |                               磁头号 |
| dl     |                             驱动器号 |
| ah     |                          02h，读数据 |
| al     |                           要读扇区数 |

将表格中的寄存器赋值后，使用 `int 13h`，就能将数据读入 es:bx 指向的缓存区。

这个参数，多难记啊。

怎么使用FAT项查找文件呢？

需要使用FAT项中的两项数据，文件名F和文件在数据区的第一块数据对应的编号N。

N有两层含义，一是FAT项在FAT中的编号，二是在数据区的簇编号。FAT项的编号的初始值是0，第零个和第一个是不使用的，从第二个开始使用。当一个文件的第一个簇号是2的时候，它在FAT中的第一个FAT项的编号是2（编号0、1的FAT项不使用），第一块数据在数据区的第一扇区（一个簇只有一个扇区）。若编号为2的FAT项的值是8，则第二块数据在数据区的第（8-2）扇区。

7654 | 3210(byte1)  7654|3210(byte2)  7654|3210(byte3)

7654|3210(byte5)  7654|3210(byte4)  7654|3210(byte3)   7654|3210(byte2)    7654 | 3210(byte1) 

ax = 4

4 * 3 = 12，12 / 2 = 6

5 *3 = 15， 15 / 2 = 7

2*3 = 6

3*3=9

一个FAT项占用1.5字节，2个FAT项占用3字节，3个FAT项占用4.5个字节，4个FAT项占用6字节，5个FAT项占用7.5字节。

FAT项编号乘以3，结果，没有啥特殊含义啊。

编号是3，乘以3，等于9。9这个数字，如何理解？编号为9的FAT项吗？

3-->4.5

2--->3

4--->6

FAT项编号是什么？

编号为0的FAT项，在FAT中的偏移量是0字节。在FAT中的偏移量（单位是扇区）是0个扇区。

编号为1的FAT项，在FAT中的偏移量是12位（1个字节 + 0.5个字节）。在FAT中的偏移量（单位是扇区）是0个扇区。

​	1. 1 * 3 / 2，商是1，余数是1。在扇区的偏移量（单位是字节）是1个字节。

编号为2的FAT项，在FAT中的偏移量是24位（3个字节）。在FAT中的偏移量（单位是扇区）是0个扇区。

5							4							 3						 2							1						0

|--------------------|--------------------|--------------------|--------------------|--------------------|

	1. 2 * 3/2，商是3，余数是0。
 	2. 3 / BytesPerSector = 0...3。在扇区的偏移量（单位是字节）是3个字节。

编号为3的FAT项，在FAT中的偏移量是36位（4个字节 + 0.5个字节）。在FAT中的偏移量（单位是扇区）是0个扇区。

根据FAT项编号获取FAT项内容，有两大难点：

1. 把FAT项编号换算成FAT项在FAT内的扇区偏移量和在扇区内的字节偏移量。
   1. 先乘以3，再除以2，怎么理解？
      1. 把FAT项编号换算成字节数。
      2. 没啥好方法。就这么理解。一个FAT项1.5个字节，编号为X的FAT项的偏移量是X * 1.5个字节。由于汇编语言不能直接完成这个运算，所以，只能转化为先乘以3再除以2。
      3. 先乘以3再除以2，商是字节数，余数是不足1字节的bit位的数量。
         1. 商是字节数，能在下面的运算（计算扇区偏移量时使用）。
         2. bit的数量，作用只是判断，FAT项的偏移量是不是刚好是1字节的整数倍。这决定了编号为X的FAT项所占用的内存是不是刚好在若干个完整的字节内。
            1. 若不在完整的字节内，从字节偏移位置，获取16位内存区域的数据，其中这16位内存区域的数据中，前4位是属于(X-1)编号的FAT项。所以，有指令 `shr ax, 4`。这是画图（就是打草稿）使用具体例子来理解的。要我自己想出来，很不容易啊。别人写出来，我都理解了这么久。
            2. 若在完整的字节内，直接获取16位内存区域的低12位的数据，那就是编号为X的FAT项的值。
2. 获得第一步数据后，根据在扇区内的字节偏移量的值的特征获取FAT项的值。



GetFATEntry

怎么理解这个函数？

它的参数是：FATEntry编号X，即在FAT区域内的偏移量。

它的返回值是：编号为X的FATEntry的那块内存区域。

首先，检查编号为X的FATEntry的字节偏移量是否占用整数个字节。

	1. 是，直接从偏移字节开始，获取1个字的内存，这个字的低12位就是目标内存区域。
 	2. 不是，从偏移字节开始，获取1个字的内存，这字的高12位就是目标内存区域。低4位是编号X-1的FATEntry的高4位。

难点是，如何检查编号为X的FATEntry的偏移量是否占用整数个字节。

一个FATEntry占用12位，即1.5个字节，FATEntry的编号乘以1.5的结果，看它们是不是整数个字节。

1. 若是整数，占用整数个字节。
2. 若不是整数，占用的不是整数个字节。

汇编语言不支持直接乘以1.5，于是，采用等效方法：先乘以3，再除以2。检查结果是不是整数。

	1. 余数不是0，不是整数。
 	2. 余数是0，是整数。

这是根据作者的代码反推出来的理解。对我而言，也不是顺其自然的。

耗费时间8个小时左右，才彻底理解了。

理解过程分为两个阶段：

一、毫无头绪，不知道为啥需要乘以3，然后再除以2。只是盯着代码看，毫无思路。

二、搜索资料，仅仅找到两篇资料。

https://www.ituring.com.cn/book/tupubarticle/26325

https://blog.csdn.net/robbie1314/article/details/5765117

这本书的内容，真的不热门。从网络资料这么少就能看出来。

资料解释得也不好。回头来看，这句话其实说出了关键点：

> 因为每个FAT表项占1.5 B，所以将FAT表项乘以3除以2（扩大1.5倍）

我开始把“乘以3再除以2”理解为计算X个FATEntry占用多少个字节。

剩下的问题是：

一、乘以3再除以2，究竟是如何产生这种运算的。

二、第一个问题中的运算的商，除以每个扇区包含的字节数，如何理解。

	1. 我一直把这个商理解为FATEntry的编号。经过第一个问题中的运算后，商是在扇区偏移量。
 	2. FAT项占用整数个字节问题。通过画图、用具体例子来理解的。34 * 57 等于多少，以前做数学题我都需要使用笔算，为啥在编程中理解问题要用心算呢？我的心算效率很低，遇到难题，容易分心，思路非常不连贯。要多用笔算。

三、es:bx，总感觉不怎么理解。通过读取软盘，包含FAT项的两个扇区被读取到了[es:bx]为初始位置的内存区域。

​	1. bx会增加吗？

## 向loader交出控制权

把loader读取到内存后，使用一个跳转指令就能把控制权从boot交给loader。

跳转指令需要内存地址，段 + 偏移量。这个内存地址，就是loader被读入到内存的位置。这个位置，在读取数据时，是可以自由设置的，不能覆盖内存中的其他数据。

```assembly
jmp BaseOfLoader:OffsetOfLoader
```



## 开始写代码

代码仓库：https://github.com/gangganghong/pegasus-os.git

代码地址：/home/cg/os/pegasus-os

最终目的，是自己独立写一个简单的操作系统。迟迟不进入正题，什么时候才能完成这个任务？

就在今日，真正开始写代码，不完成下面的功能不做其他的事情。

需要完成哪些功能呢？

1. 开发完boot。
   1. 读取loader到内存中，并执行loader中的指令。
2. 开发完简单的loader。
   1. 打印一个字符。
   2. 再打印出“hello,os"。

### boot.asm

#### 打印字符串

内存初始地址：`org 700ch`。

先不写全局描述符这些，只打印出一个字符串"hello,os!"。

打印字符串，使用BIOS中断。哪个BIOS中断？不记得了。那么，面试时如何应对别人的询问？会用和记忆是两个不同的事情。要记住，去重复看就是了。

使用`int 10h`中断。代码如下：

```assembly
mov ax, cs
mov ds, ax
mov es, ax
; 上面三句非必须
mov	ah,	13h	; 打印字符串必须使用13h
mov al, 01h	; 使用0、2、3，字符串不能正常显示，花花绿绿
; mov	bp,	BootMessage 错误写法。BootMessage不能直接赋值给bp
mov	ax,	BootMessage
mov bp, ax
; mov	cx,	BootMessageLength 错误写法。BootMessageLength不能直接赋值给cx
mov ax, BootMessageLength
mov cx, ax
mov dh,10	; 显示字符串的列坐标
mov dl,10	; 显示字符串的行坐标
int 10h

BootMessage:	db	"Hello,OS World!"
BootMessageLength	equ	$ - BootMessage
```

直接写显存，打印一个字符。代码是：

```assembly
mov ax,0B800h
mov gs, ax
mov al, 'L'
mov ah, 0ch
mov	[gs:((80*1+9)*2)], ax
```

写显存打印字符串，如何写代码？

将字符串拆分为字符，每次向显存写入一个字符，用循环显示整个字符串。



```assembly
mov	ax, 0B800h
mov gs,	ax

mov ax, BootMessage
mov es,	ax
mov	si,	ax

mov	di, 0
mov bx, 0A0h
mov ax, BootMessageLength
mov	cx,	ax

DISP_STR:
	mov	al,	es:[si]
	inc si
	mov ah,	0Bh
	mov gs:[di], ax
	add di, 2
	
	loop DISP_STR

BootMessage:	db	"Hello"
BootMessageLength	equ	$ - BootMessage
```

上面的代码是错误的，不能正确打印字符串。

下面的代码是正确的。

```assembly
; 显存从0B800h开始
mov ax, 0B800h
mov gs, ax
; di 是字符在显存中的偏移量
; (80 * row + column) * 2中，row控制字符的纵坐标，column控制字符的横坐标。
; 若增加row，di的增量需是160。若增加column，di的增量需是2。不需要思考，一瞬间就能知道这些。非要问个为什么，我还得想一想。举例，归纳法吧。
mov di, (80 * 5 + 0) * 2

; 关键语句。让si指向BootMessage这个内存位置的初始区域。
mov si, BootMessage
mov ax, BootMessageLength
mov cx, ax

DISP_STR:
	; 关键语句，将si指向的内存中的数据赋值给al。
	; 与网上的demo不同，不是[es:si]等。因为，此处的es不明，我还没找到方法获取它。[si]使用默认段地址。
	mov	al, [si]
	; 内存地址加1,让si指向下一个字节。
	inc si
	mov ah, 0Bh
	mov [gs:di], ax
	add di, 2
	
	loop DISP_STR

BootMessage:	db	"Hello"
BootMessageLength equ $ - BootMessage
```



B8000H-BFFFFH的内存空间是显存地址, 32K大小, 向这个地址写入数据就可以打印到屏幕上了。

boot有512字节，最后两个字节是魔数`0xaa55`。代码如下：

```assembly
times				510-($-$)						0
dword				0xaa55。
```

呵呵，不记得，写不出来。

魔数中的`a`是大写还是小写？大小写不敏感。

#### 从软盘中读数据

FAT12文件系统安装在软盘中还是安装在loader.bin文件中？

安装在软盘中。

怎么安装的？

1. 用bximage生成虚拟软盘。
2. 在boot.asm中写入FAT12的BPB头等信息。
3. `nasm -o boot.bin boot.asm`
4. 把boot.bin、loader.bin等写入软盘a.img中。写入工具会按照FAT12的格式要求写入数据。

BPB头等信息，内容太多，我没记住，不打算记住，直接照抄。如此，我又担心，代码究竟是我写的还是照抄的？

若我不用自己写，却对代码的了解如同我自己写过一样，那我就不自己写。

FAT12头信息：

```assembly
; 下面是 FAT12 磁盘的头
	BS_OEMName	DB 'ForrestY'	; OEM String, 必须 8 个字节
	BPB_BytsPerSec	DW 512		; 每扇区字节数
	BPB_SecPerClus	DB 1		; 每簇多少扇区
	BPB_RsvdSecCnt	DW 1		; Boot 记录占用多少扇区
	BPB_NumFATs	DB 2		; 共有多少 FAT 表
	BPB_RootEntCnt	DW 224		; 根目录文件数最大值
	BPB_TotSec16	DW 2880		; 逻辑扇区总数
	BPB_Media	DB 0xF0		; 媒体描述符
	BPB_FATSz16	DW 9		; 每FAT扇区数
	BPB_SecPerTrk	DW 18		; 每磁道扇区数
	BPB_NumHeads	DW 2		; 磁头数(面数)
	BPB_HiddSec	DD 0		; 隐藏扇区数
	BPB_TotSec32	DD 0		; wTotalSectorCount为0时这个值记录扇区数
	BS_DrvNum	DB 0		; 中断 13 的驱动器号
	BS_Reserved1	DB 0		; 未使用
	BS_BootSig	DB 29h		; 扩展引导标记 (29h)
	BS_VolID	DD 0		; 卷序列号
	BS_VolLab	DB 'OrangeS0.02'; 卷标, 必须 11 个字节
	BS_FileSysType	DB 'FAT12   '	; 文件系统类型, 必须 8个字节
```



这块头信息不能在文件的最开始位置，必需在下面代码的下面：

```assembly
jmp short LABEL_START		; Start to boot.
nop				; 这个 nop 不可少
```



为何如此？原因不明。

这是我实验的结果。当FAT12头信息在boot.asm的最开始位置时，a.img文件信息如下：

```shell
[root@localhost pegasus-os]# file a.img
a.img: DOS/MBR boot sector
```

a.img软盘若成功安装了FAT12文件系统，它的文件信息应该是下面这样的：

```shell
[root@localhost pegasus-os]# file a.img
a.img: DOS/MBR boot sector, code offset 0x3c+2, OEM-ID "ForrestY", root entries 224, sectors 2880 (volumes <=32 MB), sectors/FAT 9, sectors/track 18, serial number 0x0, label: "OrangeS0.02", FAT (12 bit)
```

##### ReadSector

这个函数从软盘中读取若干个扇区。

参数是：在软盘中的扇区号；要读取的扇区数量。

返回值：从软盘中读取的数据。

本函数内部使用BIOS中断调用`int 15h`。它需要的参数有：柱面号、磁头号、在特定柱面特定磁头的扇区号。这个扇区号的初始地址是1。这些参数都可以从在软盘中的扇区号计算得到。

读取到的数据存储在[es:bx]指向的内存中。所以，要准备一块内存存储数据。或者说，这块内存是软盘中的数据在内存中的映像区域。

用在软盘中的扇区号计算柱面号、磁头号、扇区号的方法如下：

1. 扇区号 / 每个磁道包含的扇区的数量，商是 A，余数是 B。
2. 柱面号 = A >> 1。本软盘中，只有两个磁头，即只有两个盘面。上下盘面一对编号相同的磁道组成一个柱面。
3. 磁头号 = A & 1。
4. 在本柱面本磁头的扇区号 = B + 1。在这里，扇区号的初始值是1，所以最终扇区号 = 偏移量 + 1。
   1. 这种细节问题，其实比较让我烦恼。它就是规定而已。每次去纠结这种没啥价值的小问题，实在是浪费时间。
   2. 很重要的细节。
      1. 偏移，是在上面计算出来的柱面号、磁头号上的偏移，而不是其他的柱面号、磁头号的偏移。若是后者，读取数据指定柱面号、磁头号，有何意义？用例子来理解，也证明偏移是相对于上面计算出来的柱面号、磁头号的偏移。
      2. 偏移量和编号的问题。
      3. 用具体例子来理解吧。假设每个磁道包含2个扇区。
         1. 扇区号是1。（有2个扇区）
            1. 用公式计算。商是0，余数是1。柱面号是0，磁头号是0，扇区偏移量是1，扇区号是2。
            2. 人工排列。扇区号为0的扇区是0柱面0磁头第1个扇区。扇区号为1的扇区是0柱面0磁头的第2个扇区。扇区号是2。
         2. 扇区号是2。（有3个扇区，分别是编号0、编号1、编号2的扇区）
            1. 公式。商是1，余数是0。柱面号0，磁头号1，扇区偏移量是0，扇区号1。
            2. 人工。扇区号为0的扇区是0柱面0磁头第1个扇区。扇区号为1的扇区是0柱面0磁头的第2个扇区。扇区号为2的扇区是0柱面1磁头的第1个扇区。
         3. 扇区号是3。（有4个扇区，分别是编号0、编号1、编号2、编号3的扇区）
            1. 公式。商是1，余数是1。柱面号0，磁头号1，扇区偏移量是1，扇区号2。
            2. 人工。扇区号为0的扇区是0柱面0磁头第1个扇区。扇区号为1的扇区是0柱面0磁头的第2个扇区。扇区号为2的扇区是0柱面1磁头的第1个扇区。扇区号为3的扇区是0柱面1磁头的第2个扇区。
         4. 扇区号是4。（有4个扇区，分别是编号0、编号1、编号2、编号3、编号4的扇区）
            1. 公式。商是2，余数是0。柱面号1，磁头号0，扇区偏移量是0，扇区号1。
            2. 人工。扇区号为0的扇区是0柱面0磁头第1个扇区。扇区号为1的扇区是0柱面0磁头的第2个扇区。扇区号为2的扇区是0柱面1磁头的第1个扇区。扇区号为3的扇区是0柱面1磁头的第2个扇区。扇区号为4的扇区是1柱面0磁头的第1个扇区。
      4. 理解这个公式的方法是不是不对啊？已经用这么多具体数据证明这个公式是正确的了，为何还感觉不赞同这个公式的正确性呢？我多次心算计算柱面磁头初始扇区，想找出一种不借助具体数据证明这个公式正确性的方法，没有找到。这个公式，没有多少价值。在上面耗费这么多时间，收益太小太小。要记住，不能再这样了。
      5. 在此公式耗费了四五个小时。思考，也要遵照某些规则嘛。无理由、无依据地模拟计算过程，这算哪门子的思考啊？
      6. 这个公式，计算柱面号、磁头号，都没有任何问题。问题只存在于计算在某柱面某磁头下的扇区号。搁置搁置！！！！！
      7. 想起那两段长长的读取软盘数据的汇编代码，我就觉得难。太难写了。

`int 15h` 中断的参数，我完全不记得，杂而乱。只能看书了。

```shell
# 跳过512字节查看512字节长的内容
xxd -u -a -g 1 -c 16 -s +0x200 -l 512 a.img
# 查看从文件开始512字节长的内容
xxd -u -a -g 1 -c 16  -l 512 a.img
xxd -u -a -g 1 -c 16 -s +9216 -l 512 a.img
xxd -u -a -g 1 -c 16 -s +9754 -l 512 a.img
```

通过根目录区的前面几个字符来查找目标文件的第一个簇的编号，同时也是它的第一个FAT项在FAT中的第一个FAT项的编号。编号的初始值是2。

每个根目录项的大小是32字节，一个扇区有512个字节，一个扇区能包含 (512/32) = 16 个根目录项。我们假设我们的软盘中最多只存放了14个文件。所以，只需读取一个扇区，检查14个根目录项。在这14个根目录项中没有找到目标文件，就认为目标文件不存在。

第一个根目录项，也就是根目录区的初始位置，是软盘的第19号扇区（1个引导扇区 + 18个FAT扇区）。

函数的结构是：

1. 从19号扇区开始，读取1个扇区。
2. 在这1个扇区中，读取第一个根目录项的前面几个字节，即文件名。
3. 将文件名与目标文件名逐个对比
   1. 全部相等，找到了目标文件。
   2. 不相等，将游标移动到下一个根目录项的初始位置。
      1. 将当前游标的低5位设置为0，就是下一个根目录项的初始位置。

读取了根目录区的根目录项到es:bx指向的内存区域，也知道目标文件名是“LOADER  BIN'，怎么逐个字符比较二者？

使用 es:si 和 ds:di 试试。

字符相等时，继续遍历。字符不相等时，跳出遍历。不知道怎么写代码。

查看使用FAT12的软盘的FAT区域，是不是用下面的命令？

```shell
xxd -u -a -g 1 -c 16 -s +512 -l 512 a.img
```

在软盘中，引导扇区之后是FAT区域。上面的命令，查看的就是FAT区域的开头的512个字节。

一个FAT项占用12个bit，

## 错误

```
instruction has conflicting segment overrides
```



```
Bochs is exiting with the following message:
[CPU0  ] prefetch: getHostMemAddr vetoed direct read, pAddr=0x0000000b3328
```



```
 prefetch: EIP [00010000] > CS.limit [0000ffff]
```



```
write_virtual_checks(): write beyond limit, r/w
```



```shell
<bochs:3> r
CPU0:
rax: 00000000_00000201
rbx: 00000000_00009000
rcx: 00000000_00090002
rdx: 00000000_00000100
rsp: 00000000_0000ffd0
rbp: 00000000_00007c97
rsi: 00000000_000e7ca6
rdi: 00000000_00000366
r8 : 00000000_00000000
r9 : 00000000_00000000
r10: 00000000_00000000
r11: 00000000_00000000
r12: 00000000_00000000
r13: 00000000_00000000
r14: 00000000_00000000
r15: 00000000_00000000
rip: 00000000_00007cc5
```



```shell
(0) [0x000000007cda] 0000:7cda (unk. ctxt): int 0x13                  ; cd13
<bochs:2> r
CPU0:
rax: 00000000_00000201
rbx: 00000000_00009000
rcx: 00000000_00090002
rdx: 00000000_00000100
rsp: 00000000_0000ffd4
rbp: 00000000_00007c97
rsi: 00000000_000e7ca6
rdi: 00000000_00000366
r8 : 00000000_00000000
r9 : 00000000_00000000
r10: 00000000_00000000
r11: 00000000_00000000
r12: 00000000_00000000
r13: 00000000_00000000
r14: 00000000_00000000
r15: 00000000_00000000
rip: 00000000_00007cda
```



```
00014215970p[FLOPPY] >>PANIC<< io: norm r/w parms out of range: sec#0ah cyl#71h eot#0ah head#01h
========================================================================
Bochs is exiting with the following message:
[FLOPPY] io: norm r/w parms out of range: sec#0ah cyl#71h eot#0ah head#01h
========================================================================
```



##### GetFATEntry



#### git

git 中提交代码时注释乱码问题

```shell
# 设置git 的界面编码：
git config --global gui.encoding utf-8
# 设置 commit log 提交时使用 utf-8 编码：
git config --global i18n.commitencoding utf-8
# 使得在 $ git log 时将 utf-8 编码转换成 gbk 编码。没用过。
git config --global i18n.logoutputencoding gbk
# 使得 git log 可以正常显示中文：
export LESSCHARSET=utf-8
```

Your name and email address were configured automatically based
on your username and hostname. Please check that they are accurate.
You can suppress this message by setting them explicitly:

    git config --global user.name "Your Name"
    git config --global user.email you@example.com

After doing this, you may fix the identity used for this commit with:

    git commit --amend --reset-author



#### vi/vim多行注释和取消注释

多行注释：

1. 进入命令行模式，按ctrl + v进入 visual block模式，然后按j, 或者k选中多行，把需要注释的行标记起来

2. 按大写字母I，再插入注释符，例如//

3. 按esc键就会全部注释了

取消多行注释：

1. 进入命令行模式，按ctrl + v进入 visual block模式，按字母l横向选中列的个数，例如 // 需要选中2列

2. 按字母j，或者k选中注释符号

3. 按d键就可全部取消注释

### Makefile

编译 boot.asm为boot.bin。把boot.bin写入软盘a.img。

```makefile
# 编译 boot.asm为boot.bin
nasm -o boot.bin boot.asm
# 把boot.bin写入软盘a.img
dd if=boot.bin of=a.img count=1 conv=nottruc
```

### loader.asm

Loader.asm 的任务是读入内核，开启保护模式，然后将控制权交给内核。

书中的代码还准备了gdt等。我暂时没体会到准备gdt的必要性。

本模块的开头使用`org 0x100`。`org`的用途究竟是什么？换成`org 0x200`行不行？

boot的开头是`org 0x7c00`。我的理解，写入软盘时，把boot.asm中的指令写到软盘`0x7c00`的位置。BIOS读取指令时，首先去`0x7c00`读取，读取到数据后，把数据复制到内存`0x7c00`位置。

上面的理解，似乎没问题。但是，loader开头的`org 0x100`也这样理解吗？实际上，读取loader时，遵循FAT12文件系统的读取规则，执行时，也是我自己指定的内存地址。

`org`，究竟应该怎么理解？《操作系统真相还原》里有详细的解释。有空去看看。

没感觉创建GDT的必要性。我先只完成下面的功能：

1. 读取kernel.bin
2. 开启保护模式
3. 执行kernel.bin

## kernel.asm

Loader.asm 的任务之一是读入kernel，并执行它。那么，一个最简单的kernel.asm应该怎么写？

不知道，完全不知道。老办法，记得什么，就写什么。然后看书上的代码，直至我记得应该写什么，最后开始写。

当然，这个阶段写出来的，是最简单的内核。

## 好博客

https://www.cnblogs.com/lovesqcc/p/9190544.html

https://www.cnblogs.com/lovesqcc/p/14354921.html
