# 硬盘驱动框架

## 代码位置



## 写代码的思路

### v1#获取硬盘数量

1. 新建一个进程，就像处理终端那样。
2. 新建一个进程的步骤，我忘记了。
   1. 进程表数组。
   2. 堆栈。
   3. 直接看代码吧。
   4. 已经按照其他进程的样式完成得差不多了。
3. 配置硬盘
   1. 在bochsrc中增加`ata0-master: type=disk, path="c.img", mode=flat`。
   2. 修改path为我的硬盘的路径和名称。
   3. 修改我的硬盘的用户和权限。

#### 错误

```shell
gcc -g -c -fno-builtin -o kernel_main.o main.c hd.c -m32
gcc: fatal error: cannot specify -o with -c, -S or -E with multiple files
compilation terminated.
```

#### 疑问

一、需要新建一个hd.c文件放置硬盘驱动吗？

我不知道硬盘驱动是否会用到main.c中的数据结构。若需要，如何导入？先试试吧。main.c中的文件的代码实在太多了，已经有2375行了。

不能新建。在进程表数组中需要使用硬盘驱动进程体的函数名，而这个函数名放到

二、为什么不整理main.c中的代码、把不同功能的代码放到不同的文件中？

需要耗费不少时间，我迫不及待想推进操作系统的开发进程。

### v2#分割main.h

#### Makefile中的编译动作

修改Makefile。注意下面两种不同的写法。

```makefile
hd.o:hd.c include/main.h include/string.h
        ${CC} ${CFLAGS} -o $@ $<
        
${KERNEL}:${OBJS}
        # ${LD} ${LDFLAGS} -o $@ $<
        ${LD} ${LDFLAGS} -o ${KERNEL} ${OBJS}
```

1. 输出文件和输入文件都用变量代替，在编译语句中必须使用变量表示输出文件和输入文件，不能使用`$@`和`$<`。
2. 输出文件和输入文件使用具体的文件名，在编译语句中可以使用`$@`表示输出文件，使用`$<`表示输入文件。

#### Makefile模板

```makefile
PYTHON:everything

ASM     =       nasm
CC      =       gcc
LD      =       ld
# ASKFLAGS=     -I include/ -f elf
ASKFLAGS=       -I include/ -f elf
CFLAGS  =       -I include/  -g -c -fno-builtin -m32
LDFLAGS =       -s -Ttext 0x30400 -m elf_i386

BOOT = loader.bin boot.bin
KERNEL  =       kernel.bin
OBJS    =       kernel.o string.o kernel_main.o hd.o syscall.o

image:clean everything buildimg

# everything:boot.bin loader.bin kernel.bin
everything:${BOOT} ${KERNEL}
       
buildimg:
        dd if=boot.bin of=a.img bs=512 count=1 conv=notrunc
        sudo mount -o loop a.img /mnt/floppy/
        sudo cp loader.bin      /mnt/floppy/ -v
        #sudo cp pmtest2.com    /mnt/floppy/ -v
        sudo cp kernel.bin      /mnt/floppy/ -v
        sudo umount /mnt/floppy
clean:
        rm -rvf *.bin
        rm -rvf *.o
```

#### Makefile中的include

1. 在代码中必须添加需要的文件，例如，`#include "string.h"`。

2. 在编译命令中必须添加文件，例如，`CFLAGS  =       -I include/  -g -c -fno-builtin -m32`。

3. 在编译动作中必须增加需要的文件，例如

   1. ```makefile
      hd.o:hd.c include/main.h include/string.h
              ${CC} ${CFLAGS} -o $@ $<
      ```

### v3#初始化硬盘

#### 疑问

一、时钟中断例程中，先关闭时钟中断，然后调用时钟中断处理程序，处理完毕后再打开时钟中断。硬盘中断处理例程也需要这样吗？

1. 时钟中断需要这样先关闭再打开，其实我并不知道这样做的必要性。

### v4#重构项目

#### 处理include

一些规则：

1. 把函数声明全部放在`proto.h`中。
2. `fs.h`等头文件只放数据结构，不放函数声明。
3. B头文件用到了A头文件的元素，引用时，把A头文件放在B头文件的前面。不要弄成互相引用元素的布局。

### v5#获取分区信息

#### 硬盘分区理论知识

1. 一块硬盘最多只能分为四个主分区。这是由硬盘的MBR中的DPT决定的。
2. 硬盘的DPT只有64个字节。每个分区表项需要16个字节。因此，一个DPT最多只能容纳4个分区表项。
3. 每个主分区都能作为扩展分区。但是，在四个主分区中，最多只能有一个主分区能作为扩展分区。
4. 扩展分区分为主扩展分区和子扩展分区。
5. 主分区充当扩展分区，就是主扩展分区。
6. 从主扩展分区中分割出来的扩展分区，就是子扩展分区。
7. 硬盘有一个MBR，而扩展分区（主扩展分区和子扩展分区）有一个EBR。
8. MBR和EBR的结构相同，差异在于：
   1. MBR的DPT有四个表项，EBR的DPT有二个表项。
   2. MBR的DDP的四个表项存储的数据都是主分区的元数据。
   3. EBR的DPT的两个表项存储的数据分别是：逻辑分区和下一个子扩展分区的元数据。
9. 每个主扩展分区理论上能拥有无限个子扩展分区，但是，我规定，每个扩展分区只有16个子扩展分区。
   1. 无限多个子扩展分区，我没有看到必要性。
   2. 方法一样。我能计算16个子扩展分区的情况，就能计算出32个、64个等子扩展分区的情况。
10. 我只关注分区表项中的两个数据：
    1. 起始扇区的LBA。
       1. 这个LBA并不是相对于硬盘的LBA，而是相对于本分区表所在的分区的初始地址。
       2. 因此，分区的初始扇区相对于硬盘的LBA地址 = 本分区的初始扇区相对于硬盘的LBA地址 + 分区表项中的起始扇区的LBA地址。
       3. 如果我愿意，我也可以把本数据叫做分区（表项描述的分区）在本分区的扇区偏离量。
    2. 分区的扇区数目。
11. 逻辑分区和子扩展分区A的关系
    1. 在子扩展分区中，最开始的数据是EBR。
    2. 根据EBR，我能找到逻辑分区，还能找到下一个字扩展分区B。
    3. 我的疑问是：A是否包含B？
       1. 怎么回答这个问题？
       2. A能不能包含B？
       3. A有没有包含B？
          1. 从EBR的表项，我能计算出A的范围。
          2. 比较B在不在A的范围，就知道答案了。
    4. 主扩展分区包含所有子扩展分区和逻辑分区，这很好理解。
    5. 子扩展分区之间是前后关系还是包含关系，这需要验证。
    6. 子扩展分区只包含EBR、逻辑分区，还是也包含了下一个子扩展分区，这也需要验证。我能弄清楚它，但需要时间。不弄清楚，别人问我，我当场肯定回答不出来，四五分钟内，我应该也回答不了。十分钟内，不让我分区信息，我也回答不了。

硬盘分区的理论知识，就是上面几个要点。很全面了，没有遗漏。要写出代码，只靠上面这些，我写不出来。

下一个问题，是思考遍历分区的代码应该怎么写。

#### 遍历硬盘分区

##### 主分区

1. 读取硬盘的MBR，从中获取DPT，遍历DPT。就这么简单。
2. 在遍历过程中，检查哪个分区是扩展分区。对扩展分区，进入扩展分区处理流程。

##### 扩展分区

1. 扩展分区的实质是一个链表。这个链表，我把它设计成拥有16个结点。
2. 每个扩展分区都有一个EBR。读取EBR，从EBR中获取DPT。
3. DPT的第一个表项，存储本扩展分区包含的逻辑分区。
4. DPT的第二个表项，存储下一个扩展分区的元数据。

##### 写代码

就靠上面写出来的几点，就能写出代码吗？我仍然不知道怎么写。

1. 读取MBR或EBR，应该向硬盘发送什么样的命令？

   1. LBA是多少？
   2. sector count是1。
   3. command是ATA_READ。
   4. device register中的LBA高四位、是否采用LBA、用哪个硬盘，都确定了。我打算使用固定的值。
   5. 不明确的数据是LBA。
      1. 获取MBR时，LBA是0。
      2. 获取EBR时，LBA是多少？
         1. 用主扩展分区的第一个子扩展分区为例子来计算。
         2. 第一个EBR的LBA地址
            1. 主扩展分区的LBA地址。这从MBR的DPT中获取。
         3. 从第一个EBR中获取第一个子扩展分区的LBA地址。这个LBA地址加上主扩展分区的LBA地址才是第一个子扩展分区相对于硬盘的LBA地址physical_lba1。
         4. 用这个LBA地址，能获取第一个子扩展分区的EBR EBR1。
         5. 从EBR1中，能获取第二个子扩展分区的LBA地址，即第二个扩展分区相对于第一个子扩展分区的偏移量 offset2。
         6. 第二个扩展分区的初始地址的物理LBA地址physical_lba2 = physical_lba1 + offset2。

   我只能想到这么多了。再继续想下去，其实就是心算，非常容易分心，让我非常沮丧，迟迟不能往前推进。边写边想吧。

#### 分区表结构

![image-20210621115901026](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/file-system/image-20210621115901026.png)

根据这张图，写出分区表项的struct，很容易。唯一的难点是结构体的成员的命名。

怎么读取硬盘的分区表？

向硬盘发送命令，LBA是0，sector count是1，Device Register的LBA是1。看着命令就很容易确定了。

从哪里读取分区信息，仍然是从Command Block Registers的Data寄存器读取。

发送命令前，要在硬盘空闲的时候。读取数据时，也要在硬盘空闲的时候。

读取的，其实是MBR。这句话有问题吗？没有问题。

从MBR或EBR偏移0x1BE个字节，是分区表。在MBR中的DPT中，有四个分区表项。

我关注分区表项的下面几个数据：状态、起始扇区号、分区类型、结束扇区号、起始扇区的LBA、扇区数目。

先全部打印出来吧。

#### 打印分区信息

##### 简化版

只打印主分区。

遍历MBR的分区表就行。

###### 调试

于上神的操作系统获取硬盘分区的情况如下图：

![image-20210622182156437](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/file-system/image-20210622182156437.png)

我的代码获取的主分区信息

```shell
(gdb) p partition_table[0]
$1 = {status = 0 '\000', start_head = 1 '\001', start_sector = 1 '\001', start_cylinder = 0 '\000', system_id = 131 '\203',
  end_head = 15 '\017', end_sector = 63 '?', end_cylinder = 19 '\023', start_sector_lba = 63, nr_sector = 20097}
(gdb) p partition_table[1]
$2 = {status = 0 '\000', start_head = 0 '\000', start_sector = 1 '\001', start_cylinder = 20 '\024', system_id = 5 '\005',
  end_head = 15 '\017', end_sector = 63 '?', end_cylinder = 161 '\241', start_sector_lba = 20160, nr_sector = 143136}
(gdb) p partition_table[2]
$3 = {status = 0 '\000', start_head = 0 '\000', start_sector = 0 '\000', start_cylinder = 0 '\000', system_id = 0 '\000',
  end_head = 0 '\000', end_sector = 0 '\000', end_cylinder = 0 '\000', start_sector_lba = 0, nr_sector = 0}
(gdb) p partition_table[3]
$4 = {status = 0 '\000', start_head = 0 '\000', start_sector = 0 '\000', start_cylinder = 0 '\000', system_id = 0 '\000',
  end_head = 0 '\000', end_sector = 0 '\000', end_cylinder = 0 '\000', start_sector_lba = 0, nr_sector = 0}
(gdb)
```

于上神的代码获取的主分区信息

```shell
(gdb) p part_tbl[0]
$1 = {boot_ind = 0 '\000', start_head = 1 '\001', start_sector = 1 '\001', start_cyl = 0 '\000', sys_id = 131 '\203',
  end_head = 15 '\017', end_sector = 63 '?', end_cyl = 19 '\023', start_sect = 63, nr_sects = 20097}
(gdb) p part_tbl[1]
$2 = {boot_ind = 0 '\000', start_head = 0 '\000', start_sector = 1 '\001', start_cyl = 20 '\024', sys_id = 5 '\005',
  end_head = 15 '\017', end_sector = 63 '?', end_cyl = 161 '\241', start_sect = 20160, nr_sects = 143136}
(gdb) p part_tbl[2]
$3 = {boot_ind = 0 '\000', start_head = 0 '\000', start_sector = 0 '\000', start_cyl = 0 '\000', sys_id = 0 '\000', end_head = 0 '\000',
  end_sector = 0 '\000', end_cyl = 0 '\000', start_sect = 0, nr_sects = 0}
(gdb) p part_tbl[3]
$4 = {boot_ind = 0 '\000', start_head = 0 '\000', start_sector = 0 '\000', start_cyl = 0 '\000', sys_id = 0 '\000', end_head = 0 '\000',
  end_sector = 0 '\000', end_cyl = 0 '\000', start_sect = 0, nr_sects = 0}
```

上面两段文字信息都是执行获取分区表的函数之后打印出来的分区表项。

二份的代码获取的分区信息的数据都是一样的。

使用fdisk获取的硬盘分区信息：

```shell
[root@localhost v31]# fdisk -l ./80m.img
Disk ./80m.img: 79.8 MiB, 83607552 bytes, 163296 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xd7f3d6ef

Device     Boot  Start    End Sectors   Size Id Type
./80m.img1          63  20159   20097   9.8M 83 Linux
./80m.img2       20160 163295  143136  69.9M  5 Extended
./80m.img5 *     20223  60479   40257  19.7M 99 unknown
./80m.img6       60543  90719   30177  14.8M 83 Linux
./80m.img7       90783 133055   42273  20.7M 83 Linux
./80m.img8      133119 161279   28161  13.8M 83 Linux
./80m.img9      161343 163295    1953 976.5K 83 Linux
# 
```

这本来是正确的，我却一直认为这是错误的。因为：

1. 第一个分区的system_id不是83。
2. 第三个分区表项、第四个分区表项的各项数据都是0。

回答上面的两个问题：

1. fdisk打印出来的结果中，ID就是system_id，但它却是十六进制数，83是十六进制数。`0x83` 转换为十进制是131。
   1. 我受《一个操作系统的实现》P341中“我们先是把它的分区类型（System ID）改成99h”启发，才知道fdisk获取的分区ID是十六进制数。经过验证，其他项例如Size、End等，都是十进制数。
2. 我为80m.img划分的主分区只有两个。
   1. 其中，第二个主分区被我设置成了主扩展分区。
   2. fdisk打印出来的结果中，从80m.img5到80m.img9，都是扩展分区。
   3. 80m.img5到80m.img9 都是逻辑分区，因为我在EBR的扩展分区表的第1个表项（初始序号是1）中看到表项中的扇区数和80m.img5的扇区数量相同。
   4. EBR的扩展分区表的第2个表项（初始序号是1）描述一个扩展分区，system id是5，与此吻合。

###### 调试误区

1. 分区表明明能够直接声明为`struct 分区表项 part_table[4]`，我却把它声明为`short part_table[4]`或`char part_table[64]`。
   1. 这造成了非常多不必要的麻烦，
      1. 例如，不同指针类型间转换。获取的数据，不正确。
      2. 不知道是发送给硬盘的命令不正确，还是使用指针转换发生了什么我不知道的变化。
2. 把`Memcpy`换成`Strcpy`。更是荒谬。后者会复制远超需要的数据。
3. 把存储分区表的变量数组容量修改为64，出现了Invalid Opcode。这更是系统级错误。不要随意去修改它。深不见底！
4. 想到对比于上神获取的分区信息和我获取的分区信息，是无意中开始朝着正确的方向前进。
5. 二人获取的分区信息一致，我开始找其他原因。看书，碰碰运气，看看到了那关键的一句。
   1. 知道了fdisk获取的结果中ID是十六进制。
   2. 分区信息只有两个主分区，并不是预料中的四个分区。成见害死我了！
   3. 看书也很仔细，怎么就不记得fdisk获取到的结果中ID是十六进制呢？

##### 最终版

遍历主扩展分区。

主扩展分区包含16个子扩展分区。16个子扩展分区组成一个链表。根据一个子扩展分区的EBR，能找出下一个子扩展分区的初始地址。

我要打印的，是子扩展分区的逻辑分区。

1. 在遍历硬盘分区时，遇到了主扩展分区，把这个主扩展分区当作硬盘去遍历。
2. 遍历主扩展分区，就是在遍历一个链表。
3. 怎么判断当前子扩展分区是最后一个子扩展分区？这个子扩展分区的EBR的第二个表项有什么特点？

已经写完了。

我使用了一些硬编码，不需要根据设备号计算出lba地址等，让代码简化了很多，却不损失了一些普遍性。

看于上神的代码，理解“根据设备号计算lba地址”等代码，耗费了大量大量的时间。两个“大量”，是我故意写的。

又差点误入歧途。

代码写正确后，不能在一屏打印出来，我无法通过屏幕数据观察。打印函数似乎有bug，没有正确打印出system id是5，却打印出了其他奇怪的数字。

我及时用gdb断点观察数据。在这里，又差点因为成见导致我误判数据。

在遍历子扩展分区时，EBR中的两个表项，第0个表项是逻辑分区的元数据，第1个表项是下一个子扩展分区的元数据，其中，system id是5，而不是其他。

#### 遍历分区

### v6#获取硬盘参数

#### 流程

1. 向硬盘发送获取参数的命令。
2. 通过IPC机制等待。
3. 读取硬盘参数。
4. 打印硬盘参数。

#### 操作硬盘的命令

不记得。看书上的那张图。

实际是对硬盘的块寄存器和命令寄存器的使用。

向硬盘发送命令后，从哪里读取数据？

![image-20210621133332797](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/file-system/image-20210621133332797.png)

从命令块寄存器中的0x1F0端口读取。

命令struct的成员是Features到Command。

Control Block Register的命令有什么作用？我看到向硬盘发送命令时，也向Control Block Register发送了命令，但我仍然不明白它有什么作用。

硬盘的寄存器，掌握下面几点：

1. 从Data对应的端口读取或写入数据。
2. 在向硬盘的Command Block Registers写入数据前，先向Control Block Register写入数据。
3. 向硬盘的Command Block Registers的Features到Command分别写入数据。

##### Device Register

![image-20210621164550884](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/file-system/image-20210621164550884.png)



#### IPC改造

先用固定时间的循环来看看获取硬盘参数的效果，不改造IPC。因为，改造IPC，难度大。

#### 读写硬盘的汇编的代码

看于上神的代码，然后，默写出来。本质是如此。

##### 读硬盘

使用的汇编指令：`rep insw`，读取ecx个字的数据到ds:esi处。

函数原型：`read_port(int port, char *fsbuf, int size)`

##### 写硬盘

汇编指令：`rep outsw`，把ds:esi处的ecx个字的数据写入到dx端口。

函数原型：`write_port(int port, char *fsbuf, int size)`

#### 打印硬盘参数

##### 知识点

1. 指针的使用。
2. "BA"这样的字符串修改为“AB"。
3. 硬盘参数的数据结构。书上的那张图。单位是字。

![image-20210621135649956](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/file-system/image-20210621135649956.png)

##### 流程

1. 把读取到的数据转为u16 *，方便在后面用数组的形式按字读取数据。
2. 处理两个位置的数据：
   1. 10~~~19
   2. 27~~46
   3. 把这两个区间的数据按字为单位调整一个字的数据的字节顺序。
3. 翻译60~~61区间的数据。
4. 翻译49、83的含义。

## 其他问题

远程登录虚拟机上的centos，登录成功后，出现下面的信息：

```shell
cg$ ssh root@172.16.64.132
root@172.16.64.132's password:
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Sun Jun 20 17:37:42 2021 from 172.16.64.1
```

什么意思？能使用网页连接服务器吗？

## 小结

### 2021-06-19 09:27

1. 提交/home/cg/os/pegasus-os的v4代码。
2. 弄明白/home/cg/os/pegasus-os/v31代码的用途。
3. 写了点其他东西。
4. 耗时54分。

竟然有尚未提交的的代码和未注明用途的代码。下一次，停止写代码前，在代码里建立快照：

1. 为什么停止。
2. 代码的用途。
3. 遗留了什么问题。

### 2021-06-19 09:49

/home/cg/os/pegasus-os/v31--v4分支代码有问题，因使用IPC完成get_ticks造成，仍不知道如何修改。

因搁置太久，理不清本项目的代码流程。一直在玩黑盒子。耗时15分。

### 2021-06-19 10:16

找到了IPC导致/home/cg/os/pegasus-os/v31--v4分支代码的原因：delay函数中频繁使用IPC获取ticks。

耗时20分钟。

玩黑盒子找出来的。

什么叫玩黑盒子？这里改动一下，那里改动一下，却看不懂自己写的get_ticks函数。

懒。20分钟，应该足够我看明白自己一个月前写的get_ticks函数。

我为什么不直接解决IPC问题？答案是，我试过了。难度未知，我暂时无能力解决。先跳过它，写后面的功能。

Now，I can do other things at last.

### 2021-06-19 10:16

我想把/home/cg/os/pegasus-os/v31--v4中的main.c中的函数声明和结构体等分割到main.h中。我还想用正则表达式完成。

1. 使用grep完成。失败。试过很多种正则表达式，都不行。

   1. ```shell
      1775  grep -e "\[A\][sS]*\[B\]" ./main.c
       1776  grep -e "[A][sS]*[B]" ./main.c
       1777  grep  "[A][sS]*[B]" ./main.c
       1778  grep  "^[A][sS]*[B]" ./main.c
       1779  grep  "^\[A\][sS]*[B]" ./main.c
       1826  ps -ef | grep vim
       1836  grep "\[START\][\S\s]\[END\]" ./main.c
       1837  grep "\[START\][\S\s]*\[END\]" ./main.c
       1838  grep "\[START\][Ss]*\[END\]" ./main.c
       1841  ps -ef | grep vim
       1843  grep "\[START\]([\s\S]*)\[END\]" ./main.
       1844  grep "\[START\]([\s\S]*)\[END\]" ./main.c
       1845  egrep "\[START\]([\s\S]*)\[END\]" ./main.c
       1846  grep "\\[START\\]([\s\S]*)\\[END\\]" ./main.c
       1847  grep "\\[START\\]" ./main.c
       1848  grep "\\[START\\\]" ./main.c
       1849  grep "\\[START\\]" ./main.c
       1850  grep "\\[START\\]([sS]*)\\[END\\]" ./main.c
       1851  grep "\\[START\\][sS]*\\[END\\]" ./main.c
       1853  grep "\\[START\\]([^..]*)\\[END\\]" ./main.c
       1854  grep "\\[START\\]([^.]|[.])*\\[END\\]" ./main.c
       1855  grep "([^.]|[.])*" ./main.c
      ```

   2. 没有思考，没有其他方法，在网上查查，然后一次次试。这是在玩黑盒子。

2. 使用PHP完成。试过多次后，成功。

   1. ```php
      <?php
      $content = file_get_contents('main.c');
      preg_match_all("#\[START\]([\s\S]*)\[END\]#", $content, $matchs);
      // preg_match_all("#[^.]*#", $content, $matchs);
      echo $matchs[1][0];
      ```

   2. 用上面的代码处理的代码是这样的：

   3. ```html
      [START]
      // many code
      [END]
      ```

   4. 关键点是：

      1. `#\[START\]([\s\S]*)\[END\]#`中的`[\s\S]`，不是`[sS]`。`[\S\s]`匹配所有字符，包括换行符。
      2. matchs的结构。没记住，可以看执行结果。

耗费了1小时28分。浪费时间。低效。

### 2021-06-19 11:47

把/home/cg/os/pegasus-os/v31--v4中的main.c中的函数声明和结构体等分割到main.h中。

1. 复制粘贴，删除。
2. 修改Makefile。看于上神的Makefile改造我的。看他的之前，在网上搜索了许久，没找到设置include的范例。我有示范代码，干嘛到处找未经验证的。搜索结果多是CSDN上的。

### 2021-06-19 12:27

获取硬盘数量，成功。主要难点是C指针。耗时25分。

看着书，就写出来了。

为啥这么久？

把nasm、gcc等抽取成变量后，不能编译项目。在这里耗费了最多时间。

### 2021-06-19 16:32

把string类函数从main.h中分离出来。耗时1小时41分。

做了什么？

1. 建立string.h并填充内容。
2. 修改Makefile，让它正确。

为什么这么慢？

我不懂Makefile的规则，仿照着别人的能运行的Makefile修改我的Makefile，然后测试。报错信息非常多，基本找不出有用信息。然后重新修改。

我仍然在玩黑盒子。

仿照别人的Makefile，就应该这么慢吗？

即使是仿照，也可以做得更快些。犯两次错，就应该做对，而不是反复试错。想得太少，尝试得太多。

### 2021-06-19 20:06

完成“v3#初始化硬盘”。耗时2小时44分。

完成了什么？

1. 打开8259A的第2号和第14号中断。
2. 弄明白怎么编写硬盘中断例程。
3. 主要时间消耗在第2点。这又是由于长期放置自己写的代码然后遗忘导致的。
4. 怎么弄明白的？没啥技巧，没啥条理，有点沮丧。在于上神的代码和我的代码之间看来看去，把自己弄糊涂了。最后终于弄清楚了。
   1. 我针对每个中断写例程。于上神只写由8259A主片和从片处理的宏，然后不同例程分别调用前面的宏，类似我写的系统调用例程。
   2. 进入中断后，先禁止本中断，处理完本中断后，再打开本中断。我不清楚硬盘中断是否也需要如此。
   3. 设置EOI，让中断重复发生。我几乎忘记了它的作用。从片中，需要同时设置主从片的EOI，原因不明。

把代码放得太久，却又没有准确、条理化的笔记，害我不浅！

边看边写，能比较快地推进操作系统 开发。

### 2021-06-20 05:16

搭建硬盘驱动框架。耗时40分钟。

时间消耗在处理头文件。遇到下列错误：

```shell
ld -s -Ttext 0x30400 -m elf_i386 -o kernel.bin kernel.o string.o kernel_main.o hd.o syscall.o fs_main.o
fs_main.o:/home/cg/os/pegasus-os/v31/include/main.h:125: multiple definition of `INIT_MASTER_VEC_NO'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/main.h:125: first defined here
fs_main.o:/home/cg/os/pegasus-os/v31/include/main.h:126: multiple definition of `INIT_SLAVE_VEC_NO'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/main.h:126: first defined here
fs_main.o:/home/cg/os/pegasus-os/v31/include/main.h:390: multiple definition of `user_task_table'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/main.h:390: first defined here
fs_main.o:/home/cg/os/pegasus-os/v31/include/main.h:396: multiple definition of `sys_task_table'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/main.h:396: first defined here
fs_main.o:/home/cg/os/pegasus-os/v31/include/main.h:436: multiple definition of `sys_call_table'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/main.h:436: first defined here
```



```shell
ld -Ttext 0x30400 -Map krnl.map -m elf_i386 -dynamic-linker /lib/ld-linux.so.2 -lc -o kernel.bin kernel.o string.o kernel_main.o hd.o syscall.o fs_main.o
hd.o:/home/cg/os/pegasus-os/v31/include/sys/const.h:50: multiple definition of `INIT_MASTER_VEC_NO'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/sys/const.h:50: first defined here
hd.o:/home/cg/os/pegasus-os/v31/include/sys/const.h:51: multiple definition of `INIT_SLAVE_VEC_NO'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/sys/const.h:51: first defined here
hd.o:/home/cg/os/pegasus-os/v31/include/sys/keymap.h:4: multiple definition of `keymap'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/sys/keymap.h:4: first defined here
hd.o:/home/cg/os/pegasus-os/v31/include/sys/global.h:19: multiple definition of `user_task_table'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/sys/global.h:19: first defined here
hd.o:/home/cg/os/pegasus-os/v31/include/sys/global.h:25: multiple definition of `sys_task_table'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/sys/global.h:25: first defined here
hd.o:/home/cg/os/pegasus-os/v31/include/sys/global.h:36: multiple definition of `sys_call_table'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/sys/global.h:36: first defined here
fs_main.o:/home/cg/os/pegasus-os/v31/include/sys/const.h:50: multiple definition of `INIT_MASTER_VEC_NO'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/sys/const.h:50: first defined here
fs_main.o:/home/cg/os/pegasus-os/v31/include/sys/const.h:51: multiple definition of `INIT_SLAVE_VEC_NO'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/sys/const.h:51: first defined here
fs_main.o:/home/cg/os/pegasus-os/v31/include/sys/keymap.h:4: multiple definition of `keymap'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/sys/keymap.h:4: first defined here
fs_main.o:/home/cg/os/pegasus-os/v31/include/sys/global.h:19: multiple definition of `user_task_table'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/sys/global.h:19: first defined here
fs_main.o:/home/cg/os/pegasus-os/v31/include/sys/global.h:25: multiple definition of `sys_task_table'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/sys/global.h:25: first defined here
fs_main.o:/home/cg/os/pegasus-os/v31/include/sys/global.h:36: multiple definition of `sys_call_table'
kernel_main.o:/home/cg/os/pegasus-os/v31/include/sys/global.h:36: first defined here
kernel.o: In function `hwint0.2':
kernel.asm:(.text+0x18f): undefined reference to `clock_handler'
kernel.o: In function `hwint1.2':
kernel.asm:(.text+0x1d8): undefined reference to `keyboard_handler'
kernel_main.o: In function `ReloadGDT':
/home/cg/os/pegasus-os/v31/main.c:70: undefined reference to `init_propt'
kernel_main.o: In function `init_internal_interrupt':
/home/cg/os/pegasus-os/v31/main.c:178: undefined reference to `InitInterruptDesc'
/home/cg/os/pegasus-os/v31/main.c:179: undefined reference to `InitInterruptDesc'
/home/cg/os/pegasus-os/v31/main.c:180: undefined reference to `InitInterruptDesc'
/home/cg/os/pegasus-os/v31/main.c:181: undefined reference to `InitInterruptDesc'
/home/cg/os/pegasus-os/v31/main.c:182: undefined reference to `InitInterruptDesc'
kernel_main.o:/home/cg/os/pegasus-os/v31/main.c:183: more undefined references to `InitInterruptDesc' follow
kernel_main.o: In function `kernel_main':
/home/cg/os/pegasus-os/v31/main.c:385: undefined reference to `init_keyboard_handler'
kernel_main.o: In function `sys_write':
/home/cg/os/pegasus-os/v31/main.c:529: undefined reference to `out_char'
kernel_main.o: In function `TaskTTY':
/home/cg/os/pegasus-os/v31/main.c:563: undefined reference to `init_tty'
/home/cg/os/pegasus-os/v31/main.c:564: undefined reference to `select_console'
/home/cg/os/pegasus-os/v31/main.c:568: undefined reference to `tty_do_read'
/home/cg/os/pegasus-os/v31/main.c:570: undefined reference to `tty_do_write'
kernel_main.o: In function `sys_printx':
/home/cg/os/pegasus-os/v31/main.c:718: undefined reference to `Seg2PhyAddrLDT'
/home/cg/os/pegasus-os/v31/main.c:720: undefined reference to `Seg2PhyAddr'
/home/cg/os/pegasus-os/v31/main.c:734: undefined reference to `out_char'
kernel_main.o: In function `dead_lock':
/home/cg/os/pegasus-os/v31/main.c:792: undefined reference to `pid2proc'
/home/cg/os/pegasus-os/v31/main.c:793: undefined reference to `pid2proc'
/home/cg/os/pegasus-os/v31/main.c:805: undefined reference to `pid2proc'
/home/cg/os/pegasus-os/v31/main.c:811: undefined reference to `pid2proc'
kernel_main.o: In function `sys_send_msg':
/home/cg/os/pegasus-os/v31/main.c:822: undefined reference to `pid2proc'
/home/cg/os/pegasus-os/v31/main.c:823: undefined reference to `proc2pid'
/home/cg/os/pegasus-os/v31/main.c:840: undefined reference to `Seg2PhyAddrLDT'
kernel_main.o: In function `sys_receive_msg':
/home/cg/os/pegasus-os/v31/main.c:925: undefined reference to `pid2proc'
/home/cg/os/pegasus-os/v31/main.c:926: undefined reference to `proc2pid'
/home/cg/os/pegasus-os/v31/main.c:976: undefined reference to `pid2proc'
/home/cg/os/pegasus-os/v31/main.c:979: undefined reference to `Seg2PhyAddrLDT'
kernel_main.o: In function `block':
/home/cg/os/pegasus-os/v31/main.c:1071: undefined reference to `schedule_process'
make: *** [Makefile:36: kernel.bin] Error 1
```



```shell
kernel.o: In function `hwint0.2':
kernel.asm:(.text+0x18f): undefined reference to `clock_handler'
kernel.o: In function `hwint1.2':
kernel.asm:(.text+0x1d8): undefined reference to `keyboard_handler'
kernel_main.o: In function `ReloadGDT':
/home/cg/os/pegasus-os/v31/main.c:69: undefined reference to `init_propt'
kernel_main.o: In function `init_internal_interrupt':
/home/cg/os/pegasus-os/v31/main.c:177: undefined reference to `InitInterruptDesc'
/home/cg/os/pegasus-os/v31/main.c:178: undefined reference to `InitInterruptDesc'
/home/cg/os/pegasus-os/v31/main.c:179: undefined reference to `InitInterruptDesc'
/home/cg/os/pegasus-os/v31/main.c:180: undefined reference to `InitInterruptDesc'
/home/cg/os/pegasus-os/v31/main.c:181: undefined reference to `InitInterruptDesc'
kernel_main.o:/home/cg/os/pegasus-os/v31/main.c:182: more undefined references to `InitInterruptDesc' follow
kernel_main.o: In function `kernel_main':
/home/cg/os/pegasus-os/v31/main.c:384: undefined reference to `init_keyboard_handler'
kernel_main.o: In function `sys_write':
/home/cg/os/pegasus-os/v31/main.c:528: undefined reference to `out_char'
kernel_main.o: In function `TaskTTY':
/home/cg/os/pegasus-os/v31/main.c:562: undefined reference to `init_tty'
/home/cg/os/pegasus-os/v31/main.c:563: undefined reference to `select_console'
/home/cg/os/pegasus-os/v31/main.c:567: undefined reference to `tty_do_read'
/home/cg/os/pegasus-os/v31/main.c:569: undefined reference to `tty_do_write'
kernel_main.o: In function `sys_printx':
/home/cg/os/pegasus-os/v31/main.c:717: undefined reference to `Seg2PhyAddrLDT'
/home/cg/os/pegasus-os/v31/main.c:719: undefined reference to `Seg2PhyAddr'
/home/cg/os/pegasus-os/v31/main.c:733: undefined reference to `out_char'
kernel_main.o: In function `dead_lock':
/home/cg/os/pegasus-os/v31/main.c:791: undefined reference to `pid2proc'
/home/cg/os/pegasus-os/v31/main.c:792: undefined reference to `pid2proc'
/home/cg/os/pegasus-os/v31/main.c:804: undefined reference to `pid2proc'
/home/cg/os/pegasus-os/v31/main.c:810: undefined reference to `pid2proc'
kernel_main.o: In function `sys_send_msg':
/home/cg/os/pegasus-os/v31/main.c:821: undefined reference to `pid2proc'
/home/cg/os/pegasus-os/v31/main.c:822: undefined reference to `proc2pid'
/home/cg/os/pegasus-os/v31/main.c:839: undefined reference to `Seg2PhyAddrLDT'
kernel_main.o: In function `sys_receive_msg':
/home/cg/os/pegasus-os/v31/main.c:924: undefined reference to `pid2proc'
/home/cg/os/pegasus-os/v31/main.c:925: undefined reference to `proc2pid'
/home/cg/os/pegasus-os/v31/main.c:975: undefined reference to `pid2proc'
/home/cg/os/pegasus-os/v31/main.c:978: undefined reference to `Seg2PhyAddrLDT'
kernel_main.o: In function `block':
/home/cg/os/pegasus-os/v31/main.c:1070: undefined reference to `schedule_process'
make: *** [Makefile:36: kernel.bin] Error 1
```



git config --global http.https://github.com.proxy http://127.0.0.1:1081



```shell
main.c: In function 'kernel_main':
main.c:270:23: warning: initialization of 'char *' from incompatible pointer type 'int *' [-Wincompatible-pointer-types]
  char *p_task_stack = proc_stack + STACK_SIZE;
                       ^~~~~~~~~~
main.c: In function 'TaskTTY':
main.c:562:2: warning: implicit declaration of function 'init_tty'; did you mean 'init_hd'? [-Wimplicit-function-declaration]
  init_tty();
  ^~~~~~~~
  init_hd
main.c:567:4: warning: implicit declaration of function 'tty_do_read' [-Wimplicit-function-declaration]
    tty_do_read(tty);
    ^~~~~~~~~~~
main.c:569:4: warning: implicit declaration of function 'tty_do_write'; did you mean 'sys_write'? [-Wimplicit-function-declaration]
    tty_do_write(tty);
    ^~~~~~~~~~~~
    sys_write
main.c: In function 'sys_printx':
main.c:721:12: warning: assignment to 'int' from 'char *' makes integer from pointer without a cast [-Wint-conversion]
  line_addr = base + error_msg;
            ^
main.c: In function 'sys_send_msg':
main.c:840:23: warning: initialization of 'int' from 'Message *' {aka 'struct <anonymous> *'} makes integer from pointer without a cast [-Wint-conversion]
   int msg_line_addr = base + msg;
                       ^~~~
main.c:843:28: warning: passing argument 2 of 'Memcpy' makes pointer from integer without a cast [-Wint-conversion]
   phycopy(receiver->p_msg, msg_line_addr, msg_size);
                            ^~~~~~~~~~~~~
In file included from main.c:2:
include/string.h:6:30: note: expected 'void *' but argument is of type 'int'
 void Memcpy(void *dst, void *src, int size);
                        ~~~~~~^~~
main.c: In function 'sys_receive_msg':
main.c:979:37: warning: initialization of 'int' from 'Message *' {aka 'struct <anonymous> *'} makes integer from pointer without a cast [-Wint-conversion]
                 int msg_line_addr = base + msg;
                                     ^~~~
main.c:982:25: warning: passing argument 1 of 'Memcpy' makes pointer from integer without a cast [-Wint-conversion]
                 phycopy(msg_line_addr, p_from_proc->p_msg, msg_size);
                         ^~~~~~~~~~~~~
In file included from main.c:2:
include/string.h:6:19: note: expected 'void *' but argument is of type 'int'
 void Memcpy(void *dst, void *src, int size);
             ~~~~~~^~~
gcc -I include/ -I include/sys  -g -c -fno-builtin -m32	 -o hd.o hd.c
nasm -I include/ -f elf -o syscall.o syscall.asm
gcc -I include/ -I include/sys  -g -c -fno-builtin -m32	 -o fs_main.o fs/fs_main.c
gcc -I include/ -I include/sys  -g -c -fno-builtin -m32	 -o global.o global.c
gcc -I include/ -I include/sys  -g -c -fno-builtin -m32	 -o process.o process.c
gcc -I include/ -I include/sys  -g -c -fno-builtin -m32	 -o protect.o protect.c
gcc -I include/ -I include/sys  -g -c -fno-builtin -m32	 -o console.o console.c
gcc -I include/ -I include/sys  -g -c -fno-builtin -m32	 -o keyboard.o keyboard.c
# 行不通
# ld -Ttext 0x30400 -Map krnl.map -m elf_i386 -dynamic-linker /lib/ld-linux.so.2 -lc -o kernel.bin kernel.o
ld -Ttext 0x30400 -Map krnl.map -m elf_i386 -dynamic-linker /lib/ld-linux.so.2 -lc -o kernel.bin kernel.o string.o kernel_main.o hd.o syscall.o fs_main.o global.o hd.o process.o protect.o console.o keyboard.o
hd.o: In function `TaskHD':
/home/cg/os/pegasus-os/v31/hd.c:13: multiple definition of `TaskHD'
hd.o:/home/cg/os/pegasus-os/v31/hd.c:13: first defined here
hd.o: In function `init_hd':
/home/cg/os/pegasus-os/v31/hd.c:25: multiple definition of `init_hd'
hd.o:/home/cg/os/pegasus-os/v31/hd.c:25: first defined here
hd.o: In function `hd_handle':
/home/cg/os/pegasus-os/v31/hd.c:36: multiple definition of `hd_handle'
hd.o:/home/cg/os/pegasus-os/v31/hd.c:36: first defined here
keyboard.o:/home/cg/os/pegasus-os/v31/include/sys/keymap.h:4: multiple definition of `keymap'
console.o:/home/cg/os/pegasus-os/v31/include/sys/keymap.h:4: first defined here
make: *** [Makefile:36: kernel.bin] Error 1
```



```shell
hd.o: In function `TaskHD':
/home/cg/os/pegasus-os/v31/hd.c:13: multiple definition of `TaskHD'
hd.o:/home/cg/os/pegasus-os/v31/hd.c:13: first defined here
hd.o: In function `init_hd':
/home/cg/os/pegasus-os/v31/hd.c:25: multiple definition of `init_hd'
hd.o:/home/cg/os/pegasus-os/v31/hd.c:25: first defined here
hd.o: In function `hd_handle':
/home/cg/os/pegasus-os/v31/hd.c:36: multiple definition of `hd_handle'
hd.o:/home/cg/os/pegasus-os/v31/hd.c:36: first defined here
keyboard.o:/home/cg/os/pegasus-os/v31/include/sys/keymap.h:4: multiple definition of `keymap'
console.o:/home/cg/os/pegasus-os/v31/include/sys/keymap.h:4: first defined here
make: *** [Makefile:36: kernel.bin] Error 1
```





原因是，在两个文件中都包含了`main.h`。可是，这两个文件都需要`main.h`中的函数声明。

还有两个收获：

1. 在Makefile的编译动作中，不需要添加include文件，编译C代码和汇编代码都是如此。
2. 添加多个include目录的命令是：`CFLAGS  =       -I include/ -I include/fs/  -g -c -fno-builtin -m32`。

为什么这么慢？

1. 中途看了新闻，宁波工程学院50多岁黑人外教虐杀23岁女大学生，据说是因为追求不成功。
2. 玩盒子的技术不到位。每试一次，就应该判断，不应该无脑尝试。

### 2021-06-20 06:47

修改头文件。耗时1个小时23分钟。

为啥要修改？分割main.h后，在fs/fs_main.h中使用main.h中的结构体时，出现因为头文件导致的错误。多次修改，都无效。

不大改动，已经不可能修复错误，即使修复了，在写代码的时候，随时会出现新错误。

小改动改了这么久，仍未能消除错误。

我打算怎么办？

1. 看看关于头文件引入的资料。
2. 看于上神怎么处理头文件。

### 2021-06-20 09:33

重构项目目录，耗时1小时42分。

我做了什么？

1. 仿照于上神，根据功能分割头文件和对应的函数。
2. 有些功能含糊不清，不知道分到哪个文件。例如，打印函数，头文件是`stdio.h`，实现函数的文件不能叫`stdio.c`吧。
3. 各种代码相互引用，很混乱，我不知道依赖规则是怎样的，
4. 复制粘贴。

编译时报错，非常多错误！

### 2021-06-20 14:12

修改头文件。错误实在太多。耗时40分。

复制粘贴。

### 2021-06-20 14:12

修改头文件。又失败了！失败了一次又一次。违背我对头文件的理解。耗时1小时44分。

### 2021-06-20 17:58

用简单的文件来测试头文件，却遇到了奇怪的问题。耗时1小时33分。

```shell
ld -Ttext 0x30400 -Map krnl.map -m elf_i386 -o kernel.bin main.o a.o b.o
ld: warning: cannot find entry symbol _start; defaulting to 0000000000030400
b.o: In function `say3':
b.c:(.text+0x11): undefined reference to `printf'
b.c:(.text+0x1e): undefined reference to `exit'
make: *** [Makefile:40: kernel.bin] Error 1
```

明明导入了`stdio.h`，却出现上面的错误。

我只能玩黑盒子，一次次小修改，没有思考，一次次运行，极其低效。

在bing.com搜索，没找到答案，还是谷歌靠谱。解决方法是：

```shell
ld -Ttext 0x30400 -Map krnl.map -m elf_i386 -dynamic-linker /lib/ld-linux.so.2 -lc -o kernel.bin main.o a.o b.o kernel.o
rm -f main.o a.o b.o kernel.o
```



```shell
ld -Ttext 0x30400 -Map krnl.map -m elf_i386 -dynamic-linker /lib/ld-linux.so.2 -lc
```

https://stackoverflow.com/questions/34374591/linking-an-assembler-program-error-undefined-reference-to-printf

除了最后花在搜索并修改上的5分钟，其他时间，全部在玩黑盒子中浪费。

### 2021-06-20 22:49

终于彻底重构好了头文件。耗时2个小时52分。

一共耗费了11个小时54分。

我在做什么？

1. 复制、剪切文件、运行。由于代码不是很少，需要用git在物理机和虚拟机同步代码，一共周期比较长，至少，心理感觉很长。

2. 我没有什么思考，准确说，思考很少。我原以为头文件问题非常简单，哪里知道这么难处理。我没啥这方面的经验，仅有的经验对解决问题无效。

3. 自己写简单代码来摸索头文件的用法，却遇到了`ld`问题。又玩了很久黑盒子。谷歌解救了我。只恨没有早点用谷歌搜索。

4. 怎么解决的？

   1. 对照于上神的头文件处理方法，处理我的代码。

      1. 仍然不懂得原理。

   2. 在网上找到了这句：

      1. ```makefile
         ${KERNEL}:${OBJS}
                 # 1. $^ 非常有用。
                 # 2. 它解决了错误：
                 # 3. keyboard.o:/home/cg/os/pegasus-os/v31/include/sys/keymap.h:4:
                 #       multiple definition of `keymap'
                 ${LD} ${LDFLAGS} -o $@ $^
                 #${LD} ${LDFLAGS} -o ${KERNEL} ${OBJS}
         ```

      2. 是这句`${LD} ${LDFLAGS} -o $@ $^`中的`$@ $^`。

      3. 早点尝试就好了。

花的时间实在太长了。不停地复制、剪切、运行，偶尔搜索。

Makefile是个工具，要么系统地看看它的用法，要么多尝试几种方法。

我使用的是后者。拟定几种不同的方法，一种一种地尝试。即使有十种方法，大概只需要十个小时。

我统计过，第一次大规模改动项目目录，需要四十分钟左右。后面的改动要小得多。所以，用暴力方法，也不需要十个小时。

那么，我为啥用了这么久？我在用一种方法反复尝试。即使只有一道很简单的数学题，做100次，也会耗费很多时间。

头文件问题，我这次遇到的，分为两类：一种是函数或变量未定义，一种是重复定义。

重复定义用`$@ $^`解决，未定义只需仔细安排好函数、变量的顺序就能解决问题。

看了一次正确的头文件，总结不出有什么规则。

### 2021-06-21 08:55

电脑硬盘空间不够，内存空间也不够。原因不明。把已知的能删除的文件全部删除后，硬盘空间也没有恢复到几天前那么大。我又没有新增文件到电脑上。

我把开发操作系统的虚拟机上的操作系统项目中的硬盘、软盘都删除了，把/var/cache也删除了。

要到的命令有：

```shell
# 显示当前目录下的一级子目录的大小，显示单位是人类可识别的MB
du -h --max-depth=1
# 删除路径名包含img、但是不包含e的文件
fin ./ -name "*.img" | grep -v "e" | xargs rm -rvf
```

使用范例

```shell
find ./ -name "80m*" | grep -v "e" | xargs rm -rvf
[root@localhost osfs10]# du -h --max-depth=1
3.9M	./.git
83M	./a
83M	./b
83M	./c
85M	./d
85M	./e
420M	.
```

耗时1小时30分。

为啥耗费了这么久？

1. 搜索解决方法。搜索到了`du -h --max-depth=1`是一个标志，标志我找到了正确的解决方法。
2. 查看虚拟机上的文件。
3. 确认虚拟机上的centos8中的/var/cache能不能删除。
4. 逐个删除不需要冗余的文件（例如操作系统功能中的80m.img)；批量删除。
5. 重新提交代码。
6. 能不能再快些？那只能省略3、4、1。



> `/var/cache` is intended for cached data from applications. Such data is locally generated as a result of time-consuming I/O or calculation. The application must be able to regenerate or restore the data. Unlike `/var/spool`, the cached files can be deleted without data loss. The data must remain valid between invocations of the application and rebooting the system.
>
> Files located under `/var/cache` may be expired in an application specific manner, by the system administrator, or both. The application must always be able to recover from manual deletion of these files (generally because of a disk space shortage). No other requirements are made on the data format of the cache directories.

> Unlike `/var/spool`, the cached files can be deleted without data loss. 

https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch05s05.html

上面这段英文，说，/var/cache，能被删除。

练习linux命令

```shell
# 找出bochs进程、排除grep进程并且只获取进程PID，然后用kill杀死这个进程
ps -ef | grep bochs | grep -v grep | awk '{print $2}' | xargs kill 9
```

### 2021-06-21 12:20

工作内容：v6#获取硬盘参数。耗时41分。

做了什么？

1. 理清大概思路。
   1. 先向硬盘发送命令。
   2. 等待。
      1. 为啥要等待？硬盘读取数据需要时间。不可能发送了命令，然后就能读取到数据了。
      2. 代码等待，需要手工实现。如果不手工实现，执行读取数据的指令将获取不到数据。
   3. 读取第0块硬盘的LBA是0的扇区。
   4. 把读取到的数据
2. 看于上神的代码，弄清怎么读取分区表，同时，看不懂他的代码：发送读取命令后，在读取数据前，调用了`interrupt_wait()`。

我弄错了，其实我在v5要做的是获取硬盘参数，却误以为是获取分区信息。又耗费11分。

> 可以边看书边写。写完之后，却实在感觉不踏实，再多看几次，甚至，我可以从头开始写一次。

理解`u16* hdinfo = (u16*)hdbuf;`，又耗费了几分钟。

这句的作用是，在`char * p = (char*)&hdinfo[iinfo[k].idx];`中把hdinfo当数组用，每个元素的长度是一个字。

### 2021-06-21 12:20

工作内容：v6#获取硬盘参数。耗时41分。中途不想干了，枯燥无味，看了一会公众号和凤凰网。

上个小结在看硬盘分区内容，我却误以为在看硬盘参数内容。v6的主要内容都是在这半个多小时写完的。

有点枯燥，一想到还有那么多工作，不知道做完了这个东西有没有用，我就不愿继续写操作系统了。

还算顺利。

我的感受：比前几天逐行看代码复习笔记，效率高，进度快。

第一次看懂了书中的内容和代码的主要流程和逻辑后，我其实就可以直接边写边看了。中途再次逐行看代码，收获很小。

能逐行看懂每个细节，当然好啊。可我花了那么偶时间，总有疑问，疑问还越来越多。

所以，我应该逐行看代码，但不应该在看不懂的地方耗费一天的时间。看不懂，先跳过。

开发IPC功能，我看不懂中断的作用，现在做到硬盘功能，反而理解了前面的疑惑。

适当囫囵吞枣，是正确的。

### 2021-06-21 16:22

写完了v6中的read_port、write_port，能通过编译，正确性未知。不知道怎么测试。

耗时30分钟。很顺利。

做了什么？

1. 想包含这两个函数的文件名、放哪里。纠结。这并不重要。
2. 写函数，就是对`rep outsw`和`rep insw`的使用。
3. 修改Makefile。

### 2021-06-21 16:22

写完了v6中的Device Register生成宏MAKE_DEVICE_REGISTER、hd_cmd_out。

耗时2小时7分。

没有难度，复制照抄寄存器端口常量等。

写完了读硬盘参数的代码，初步测试了一下，由于没有写出打印参数的代码，只能用gdb断点打印。

读取硬盘参数失败，原因未知。

执行完read_port后，cmd的数据会被清空。

接下来，就考验调试能力了。

### 2021-06-21 19:10

读取硬盘参数调试，失败，玩黑盒子，改动一下，测试，反复如此。

错误现象：在hd_cmd_out后的代码没有执行。

还有，在loader.asm中设置的8259A的硬盘和级联位，无效。在init_hd中设置的有效。

耗时39分。实在太浪费时间了！

### 2021-06-21 20:24

读取硬盘参数，有反应了。之前，卡住，不能往下执行。

我怎么做的？

对比于上神的代码和我的代码执行过程中的cmd数据。

再对比到端口读取函数read_port，发现，我的函数和于上神的不同。修改成和他的一样，读硬盘操作有反应了。

自己粗心大意弄错了`insw`。

> INSW           15,pm=9*/29**  Input word from port DX into ES:(E)DI

耗时30分。

大部分时间消耗在测试。

### 2021-06-21 22:45

读取硬盘参数并且解析序列号成功。耗时2小时22分。大部分时间消耗在修改、编译、运行，基本是在玩黑黑子，没有思考。



```c
// code-A
// 这句会导致invalid opcode，为什么？实在太令人费解了！
unsigned short *hdinfo = (unsigned short *)buf;
```



```c
// code-B
// 延迟一会。必须延迟一会。
// 频繁使用IPC，所以不能使用。
// milli_delay(5000);
delay(250); //导致invalid opcode
Printf("%s\n", "delay over");
```



```c
// code-C
void hd_handle()
{
        Printf("%s\n", "HD handle is running!");
        Message msg;
        send_rec(RECEIVE, &msg, ANY);
        unsigned int type = msg.type;
        unsigned int source = msg.source;

        switch(type){
                case HD_DEV_OPEN:
                        Printf("%s:%d\n", "Open HD", source);
                        hd_identify(0);
                        break;
                default:
                        Printf("%s\n", "Unknown Operation");
                        break;
        }

        msg.source = 2;
        // ipc存在问题，使用频繁，会导致IPC异常，所以，我暂时注释主句。
        //send_rec(SEND, &msg, source);
}
```



这三个问题，有两个和存在bug的IPC有关。

```c
// code-3
#include <stdio.h>

int main(int agc, char *argc[])
{
        char name[32] = "How are you?Fine,thank you.";
        unsigned short *buf = (unsigned short *)name;
        printf("buf = %s\n", buf);

        return 0;
}
```



code-3和code-1本质相似，但是code-3能正常运行，code-1却不行，究竟是为什么？

我为什么花了这么多时间才找出问题？

问题太多了。交织在一起。invalid opcode、ipc问题、硬盘和硬盘驱动互相等待的关系。

在发送硬盘命令的每条语句间需要打印语句才能正常。呵呵，这种表象，不看于上神的代码，我想破脑袋都想不明白是什么原因。

我认为，是IPC没有让硬盘驱动在发送硬盘命令后阻塞导致的。我简化方式是在发送硬盘命令后空转一段时间。

应该都是IPC导致的问题。

我没有能力现在去解决IPC问题，先搁置，写完硬盘的初级操作再说。等到后面，不解决IPC不行，再攻克IPC!

大部分时间都在玩黑盒子啊（小修改、编译、运行），体力劳动。

### 2021-06-21 23:58

打印硬盘参数，成功了。

![image-20210621235849665](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/file-system/image-20210621235849665.png)

easy.

cost time:26 minutes.

难点：

```c
char capabilitie_lba[4] = (support_lba == 1 ? "Yes": "No");
```

非法语句。

### 2021-06-22 11:40

v5理论知识。比较顺畅。耗时31分钟。

边看边写，是正确的方式。难度降低，进展快。

### 2021-06-22 13:45

硬盘分区，不是我之前想的那样。烦恼。看了于上神的代码，我又明白了硬盘分区的真正情况。

呵呵，我这是在背诵吗？我需要依靠看别人的代码才能知道知识。这让我很烦恼。

不想继续写了。中途我看了比较长时间的其他资讯、视频。

耗时33分钟。一行代码没有写。

视频、公众号信息、凤凰网、焦虑之事、硬盘分区和我之前想的不一样，都让我烦恼。

一心不二用。边学习边看无用资讯，不如不学。

硬盘分区，有点麻烦，认真写笔记，彻底弄明白他。

### 2021-06-22 13:45

v5#硬盘分区理论知识，写要点和心算，基本弄清楚了。

唯一的纠结点是：子扩展分区之间的关系。仍然没有弄清楚。

耗时41分。

### 2021-06-22 15:27

v5#遍历分区的代码思路。耗时37分。

只能想到这么多，继续把所有代码都想得很清楚，十分困难，沮丧，分心。

边写边想。

### 2021-06-22 16:52

v5#遍历硬盘主分区，打印主分区，代码写完。耗时1个小时22分。

在写代码的过程中，之前不知道怎么设计参数的函数，都慢慢清晰起来。

初步测试，数据不正确。

测试，耗费了最多时间。

### 2021-06-22 19:01

#v5变量硬盘主分区调试正确。耗时1小时31分。耗时3个小时，才完成这个小功能。

什么感受？沮丧，有点绝望，但最后，又是“柳暗花明又一村”。

为啥花了这么多时间？

1. 缺少正确的知识，把正确结果当错误的。有眼不识泰山。耗时最多。
2. 常识性错误。说不出原因。如此轻松、悠闲，都会写出这样错误的代码。在外996，头昏脑涨，更容易犯这种错误。
   1. 分区表明明能够直接声明为`struct 分区表项 part_table[4]`，我却把它声明为`short part_table[4]`或`char part_table[64]`。

### 2021-06-22 21:24

#v5打印分区信息#最终版，写完了代码，测试通过。耗时1个小时14分。

基本思路，不能心算出来，写着写着就明朗起来。

两个关键点：

1. 遇到主扩展分区时，把主扩展分区当作硬盘一样，遍历主扩展分区，即调用分区函数partition。出现了递归。
2. 在遍历子扩展分区链表时，先获取分区表。分区表的第0个表项描述逻辑分区，第1个元素描述下一个子扩展分区。
3. 分区表项的system id的特殊值。
   1. 当分区表的表项的system id是0时，这个表项不指向任何分区。遍历子扩展分区链表的终止条件是，当前子扩展分区的分区表的第1个表项的system id是0。这表示当前子分区是最后一个子扩展分区。
   2. 0x5，描述扩展分区，或者，指向扩展分区。
   3. 0x83，Linux分区。

接下来，做什么？

1. 让终端屏幕随着打印内容增加而自动向下滚屏。
2. 把获取到的硬盘分区信息存储到合适的struct中。