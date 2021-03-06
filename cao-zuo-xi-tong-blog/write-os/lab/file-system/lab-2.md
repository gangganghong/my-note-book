# 文件系统

## 代码位置

/home/cg/os/pegasus-os/fs/v1

## v1#在硬盘上制作文件系统

### 我的设计#概况

文件系统的每个要素依次从硬盘的M处开始分布如下：

1. boot sector。不需要填充任何数据，只需为它留一个扇区即可。
2. super block。很复杂的结构。我不理解它存在的必要性。
3. inode-map。使用bit记录inode-array中每个inode的使用情况。
4. sector-map。使用bit记录每个扇区的使用情况。
5. inode-array。记录文件的元数据。
6. 数据区
   1. 根目录区。有多少个文件，就有多少个目录项。
      1. 根目录项
      2. dev_tty0
      3. dev_tty1
      4. dev_tty2
   2. 文件数据

M是多少？是硬盘的第2个主分区的第1个逻辑分区。这是人为规定。

### 我的设计#流程

用`open`来理解文件系统。

1. 文件名是关键线索。
2. 根据文件名找到文件的元数据。
   1. 存储文件的元数据的有inode、文件描述符、目录项。
   2. 很纠结。刚才没有条理地想了几分钟。思考，要有条理性、要逻辑严谨。否则，那是在瞎想。
   3. inode存储文件的初始LBA、分配给文件的扇区、文件当前已经使用的扇区数量。它当然有必要存在。
   4. 目录项存储文件的文件名。仅此一条，它也应该存在。
   5. 能把文件名存储在inode吗？如此，连目录项也不需要存在了。
      1. 要回答这个问题，方法是，试一试，把文件名存储在inode能否满足所有需求。
      2. 遍历inode-array，根据文件名找到目标inode，能做到。
      3. 能不能支持多级目录？能。为inode增加一个成员parent_inode_nr。
      4. 目录项好像也不需要了。
3. 读写文件，有权限限制，例如只读、只写（不存在只写吧）、可读可写。
4. 不同进程打开同一个文件，能设置不同的权限。如果依靠inode描述文件操作权限，不能满足这种要求。
5. A进程打开文件，权限是只读；B进程打开文件，权限是可读可写。
6. 如果只用一个成员描述文件的权限，这显然做不到。非要在inode中描述文件的权限，会设计出非常复杂的数据结构。
7. 这是完全没有必要的。添加一个文件描述符，问题会简化很多。
8. 仅仅因为“文件操作权限”，文件描述符非常有必要存在。
9. 文件描述符是操作文件的进程和目标文件的inode之间的桥梁。
   1. 一个进程能够操作很多个文件，因此，它可以拥有很多个文件描述符。
   2. 存储多个文件描述符，用数组很合适。因此，在进程表中新增一个成员filp，存储文件描述符。
   3. 文件描述符至少有两个成员：文件操作权限和描述文件的inode。
   4. 文件操作的位置，也应该记录在文件描述符中。和文件操作权限一样，不适合存储在inode中。
   5. 文件操作的并发性，先不考虑吧。这是一团迷雾。我很缺乏这方面的知识，强行思考，会导致”思而不学则殆“，陷入低水平得到思考。
10. 再想想，根目录有必要存在吗？
    1. 我为什么偏向于论证有必要存在根目录？
    2. 因为，于上神和其他专家设计的文件系统都存在根目录。他们这样做，一定有原因。
    3. 因为要记录文件操作的权限、文件操作的位置，所以，必须有文件描述符。
    4. 确实，一定量的知识，是思考的基础。以”必要性“为例，我知道文件操作的权限、文件操作的位置，在假设过程中，就能够想到这两种，就知道了把它们存储在inode中很不合适。根目录有存在的必要性吗？思考方法，就是假设、推理。可我找不到从哪方面去推理。就这样吧。
11. 超级块有必要存在吗？
    1. 超级块的成员非常多。
    2. 我没有感觉到它有存在的必要。先不要它。只为它预留空间，等体会到它的必要性时再增加。

#### 初始化文件系统

把文件系统写入到硬盘的第1个主分区的第1个逻辑分区。

1. boot sector。只留一个扇区。
2. super block。只留一个扇区。
3. inode-map。
   1. 怎么处理？
   2. 第0个bit，不使用。
   3. 从第1个bit开始，记录inode-array中的inode的使用情况。
   4. inode-map[1]记录inode-array[0]的使用情况，inode-map[2]记录inode-array[1]的使用情况。
   5. inode-map占用几个扇区？
4. sector-map。
   1. 怎么处理？
   2. sector-map[0]记录硬盘的第1个主分区的第1个逻辑分区还是记录数据区的第1个逻辑分区？
5. inode-array。
   1. 预置四个inode
      1. 根目录的inode
      2. 终端0的inode
      3. 终端1的inode
      4. 终端2的inode
   2. inode-array占用几个扇区？
6. 数据区
   1. 第一部分，根目录
      1. 根目录中存储目录项。
      2. 每个文件需要一个目录项。
      3. 根目录自身也需要目录项吗？
   2. 第二部分，其他数据

#### 读写硬盘

文件系统是一个数据结构，构建好数据结构，还要把这个数据结构安装到硬盘上。”安装“就是写到硬盘上。

读写硬盘是硬盘驱动的功能，在文件系统进程中，通过IPC请求硬盘驱动读写硬盘。

> 把硬盘驱动放到本篇来写。连贯。

##### 打开硬盘

仅仅在硬盘信息中新增一次打开的记录。

##### 读写硬盘

硬盘操作，需要提供：

1. 设备号。通过主设备号找到硬盘驱动，通过次设备号找到数据基址。
2. 位置。相对于数据基址的偏移量。
3. 数据。写操作才有数据，读操作没有。
4. 长度。读多少数据，写入多少数据。

###### 读

读取数据的长度。实际读取的长度是：min(要读取的长度，硬盘上的剩余的数据的长度)。

###### 写

实际写入的长度是：min(要写入的长度，硬盘上剩余空间的长度)。

#### 关闭硬盘

在硬盘信息中减去一次打开的记录。

#### 创建文件

> 卡壳了。卡壳就容易分心，想了一些琐碎的往事。
>
> 遇到这种情况，就尽快看书上是怎么做的。

1. 在inode-map中找到一个空闲的inode，分配固定数量的扇区。
2. 文件描述符在哪里产生？不知道。

自己设计文件到处为止。再继续下去，就是“思而不学则殆”，低水平思考，瞎想。

### 于上神的设计

#### 数据结构

##### super block

用数组存储super block，只有第一个元素填充了数据。

```shell
(gdb) p super_block[0]
$5 = {magic = 273, nr_inodes = 4096, nr_sects = 40257, nr_imap_sects = 1, nr_smap_sects = 10, n_1st_sect = 269, nr_inode_sects = 256,
  root_inode = 1, inode_size = 32, inode_isize_off = 4, inode_start_off = 8, dir_ent_size = 16, dir_ent_inode_off = 0,
  dir_ent_fname_off = 4, sb_dev = 800}
(gdb) p super_block[1]
$6 = {magic = 0, nr_inodes = 0, nr_sects = 0, nr_imap_sects = 0, nr_smap_sects = 0, n_1st_sect = 0, nr_inode_sects = 0, root_inode = 0,
  inode_size = 0, inode_isize_off = 0, inode_start_off = 0, dir_ent_size = 0, dir_ent_inode_off = 0, dir_ent_fname_off = 0, sb_dev = 0}
(gdb)
```

`inode_size = 32`，可以看出，inode的长度是32，而在代码中inode的长度是44。44和32之间的差距，正好是代码中inode的最后三个成员的长度。

超级块中的inode_size是inode在硬盘中的长度，而代码中inode的长度是在内存中的长度。代码中的inode的最后三个成员只存在于内存中。

`nr_smap_sects`的值不需要和`nr_sects`相等，只需不小于`nr_sects`。

`n_1st_sect = 269`，怎么计算出来的？

1. boot sector，占用1个扇区。
2. supber block，占用1个扇区。
3. Inode-map，占用1个扇区。
4. sector-map，占用10个扇区。
5. inode-array，占用256个扇区。
6. 小计：13 + 256 = 269。
7. 269，正好是`n_1st_sect`的值。
8. 这说明，`n_1st_sect`从数据区开始统计。根目录也在数据区。
9. 根目录占用多少空间？
   1. 我以为，根目录应该为所有文件大小预留空间。
   2. 一个目录项占用16个字节，4096个根目录占用 `4096*16/512 = 8*16`个扇区。

超级块成员结构太多了。

```shell
[root@localhost v1]# fdisk -l 80m.img
Disk 80m.img: 79.8 MiB, 83607552 bytes, 163296 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xd7f3d6ef

Device     Boot  Start    End Sectors   Size Id Type
80m.img1            63  20159   20097   9.8M 83 Linux
80m.img2         20160 163295  143136  69.9M  5 Extended
80m.img5   *     20223  60479   40257  19.7M 99 unknown
80m.img6         60543  90719   30177  14.8M 83 Linux
80m.img7         90783 133055   42273  20.7M 83 Linux
80m.img8        133119 161279   28161  13.8M 83 Linux
80m.img9        161343 163295    1953 976.5K 83 Linux
```

`nr_sects = 40257`，和`80m.img5   *     20223  60479   40257  19.7M 99 unknown`中的Sectors相同。

`nr_sects`是什么？是文件系统占用的全部扇区数量，包括文件系统中用来存储文件数据的数据区域占用的文件的数量。

那么，剩余的`80m.img6`、`80m.img7`、`80m.img8`、`80m.img9`在被文件系统覆盖了吗？没有。我们的文件系统只安装在`80m.img5`这个逻辑分区。

##### inode

inode中有个成员是`char _unused[16]`。inode的大小是44个字节。44个字节不能被被512整除。这意味着一个扇区容纳若干个inode后还有剩余的不足一个inode的空间。

`char _unused[16]`的作用是对齐，这是啥意思？

```c
struct inode{
  	char i_mode;
  	int i_size;		// 文件大小
  	int i_start_sector;	// 文件的第一个扇区的偏移量还是绝对LBA地址？
  	int i_nr_sectos;		// 分配给文件的扇区数量。文件的大小是固定的。
  	char _unused[16];		// 没有作用，字节对齐使用。有必要手工对齐吗？
  
  	// 只在内存中的成员
  	int i_dev;		// 设备号
  	int i_cnt;		// 被多少个文件描述符共享
  	int index;		// inode在inode-map中的索引
  	
};
```

我不赞同这种命名方式。inode的成员不需要再加上inode作为前缀，正如，数据库中，表的字段名不需要加上表名作前缀。

文件的第一个扇区的偏移量还是绝对LBA地址？都行。只是，有优劣之分。存储绝对LBA地址，只需要计算一次；存储偏移量，每次都需要计算。

占用的空间有差别吗？如果存储二者都使用`int`数据类型，没有差别。用`int`存储8和存储8888，占用的内存空间都是4个字节。

我不知道怎么计算i_start_sector。

很好计算。

1. 第1个文件的i_start_sector的偏移量是0*i_nr_sectos。
2. 第2个文件的i_start_sector的偏移量是1*i_nr_sectos。
3. 第2个文件的i_start_sector的偏移量是2*i_nr_sectos。
4. 第n个文件的i_start_sector的偏移量是(n-1)*i_nr_sectos。
5. n是什么？
   1. 是根目录中目录项的序号（初始值是1）。
   2. 是inode-map的索引（初始值是0）。
   3. 是inode-array的索引（初始值是0）。

##### dir_entry

```c
struct dir_entry{
  	int inode_nr;
  	char name[MAX_FILENAME_LEN];
};
```

so easy.

没啥可以深究的。有多少个文件，在根目录中就有多少个目录项。

在文件系统中寻找文件的流程如下：

1. 搜索关键词是filename。
2. 遍历根目录，目录项的name是filename的目录项是目标目录项。
3. 根据目录项中的inode_nr找到目标inode。

文件描述符有什么用？在上面的流程中，没有看到文件描述符的参与。

##### 文件描述符

```c
struct file_desc{
  	int fd_mode;		// 操作权限，例如：可读、可写
  	int fd_pos;			// 文件操作位置
  	int fd_cnt;			// 有多少个进程共享这个文件描述符
  	int fd_inode;	// 文件描述符对应的inode
};
```

唯一的疑问：`fd_cnt`。

进程还能共享文件描述符吗？

难道不是一个进程打开一个文件就产生一个文件描述符吗？

#### 我的设计#编码建立文件系统

> 在“于上神的设计”中混入了“我的设计”，不妥。
>
> 不管它，我先自己设计一番。另外用一个小节写于上神的设计。

把文件系统这个数据结构写入到硬盘。

> 建立文件系统，有两种做法。一种是，我先想想，应该怎么做。第二种是，我先看于上神的代码，然后复述出来，照着做。
>
> 第二种，很简单，能推进速度。第一种，可能也并不困难，但是，我觉得很困难。
>
> 在“我的设计”中，我根据记忆写出了我的文件系统方案。在看于上神的代码时，我没有看自己的成果。所以，我怀疑，先自己想一次，好像没有作用。

向硬盘写入数据，需提供下列数据：

1. 设备号
   1. 主设备号：选择驱动程序。
   2. 次设备号：硬盘操作的基地址。
2. 写入数据的初始位置。
3. 要写入的数据。
4. 要写入的数据的长度。

把下面的后五个数据结构写入硬盘时，都需要提供上面的4个数据。

##### boot sector

没有数据结构。只预留一个扇区。

##### super block

预置数据。然后写入一个扇区中。

不知道超级块有什么作用。

难点是，向硬盘写入数据。

文件系统通过IPC调用硬盘驱动来完成。

超级块的每个成员是什么值，我不记得。在写的时候再设置吧。

向硬盘写入数据，需提供下列数据：

1. 设备号。
   1. 主设备号：选择驱动程序。
   2. 次设备号：硬盘操作的基地址。第16个逻辑分区。
2. 写入数据的初始位置。第1个扇区（初始序号是0）。
3. 要写入的数据。`struct super_block super_block`。
4. 要写入的数据的长度。`sizeof(struct super_block)`。

##### inode-map

记录inode-array中每个inode的使用情况。inode-map的第1个bit是保留位，第2个bit记录数据区的第1个inode的使用情况，第3个bit记录第2个inode的使用情况，第N个bit记录第N-1个inode的使用情况。

只需要设置第1个字节的`第0~~第4`个bit是1，然后把这个扇区写入硬盘。

1. 设备号。
   1. 主设备号：选择驱动程序。
   2. 次设备号：硬盘操作的基地址。第16个逻辑分区。
2. 写入数据的初始位置。第2个扇区（初始序号是0）。
3. 要写入的数据。
   1. 本扇区的第1个字节的前4个bit设置成1，后4个bit设置成0。
   2. 本扇区其余511个字节的所有bit全部设置成0。怎么写代码实现？
4. 要写入的数据的长度。`sizeof(struct super block)`。

##### sector-map

记录数据区中每个扇区的使用情况。sector-map的第1个bit记录数据区的第1个扇区的使用情况，sector-map的第2个bit记录数据区的第2个扇区的使用情况，sector-map的第3个bit记录数据区的第3个扇区的使用情况，sector-map的第N个bit记录数据区的第N个扇区的使用情况。

把第1个字节的第0个到第3个bit设置成1，然后把这个扇区写入硬盘。

1. 设备号。
   1. 主设备号：选择驱动程序。
   2. 次设备号：硬盘操作的基地址。第16个逻辑分区。
2. 写入数据的初始位置。第3个扇区（初始序号是0）。
3. 要写入的数据。
   1. 本扇区的第1个字节的前3个bit设置成1，后5个bit设置成0。
   2. 本扇区其余511个字节的所有bit全部设置成0。怎么写代码实现？
   3. sector-map的所有其他扇区都设置成0。还有多少个扇区？
4. 要写入的数据的长度。是sector-map的所有扇区的字节数量吗？
5. sector-map究竟记录数据区的每个扇区的使用情况还是记录安装了文件系统的分区的每个扇区的使用情况？
   1. 暂时无从得知。

##### inode-array

文件系统能存储多少个文件，是固定的。因而，inode-array中的inode的数量也是固定的。

共有多少个inode？

每个inode占用32个字节，初始化文件系统时有4个inode，占用128个字节。

怎么初始化inode-array？

在inode-array区域写入的数据是：`struct inode inodes[4]`。

1. 设备号。
   1. 主设备号：选择驱动程序。
   2. 次设备号：硬盘操作的基地址。第16个逻辑分区。
2. 写入数据的初始位置。1(boot sector)+1(super block)+inode-map占用的扇区的数量+sector-map占用的扇区的数量
3. 要写入的数据：`struct inode inodes[4]`。
4. 要写入的数据的长度。~~是sector-map的所有扇区的字节数量吗？~~
   1. 莫名其妙的错误。要写入的数据的长度是`4 * sizeof(typedef struct inode)`。

##### 根目录

有四个预置的目录项，分别是：根目录自身、终端0、终端1、终端2。

在根目录区区域写入的数据是：`struct dir_entry dir_entries[4]`。

##### 数据区

#### struct和数据对齐

```shell
#include <stdio.h>

struct A{
        char a;
        int b;
        short c;
};

int main(){
        struct A a;
        printf("A: %ld\n", sizeof(a));
        return 0;
}
```

执行结果：

```shell
[root@localhost c]# gcc -o align-demo align-demo.c
[root@localhost c]# ./align-demo
A: 12
[root@localhost c]# gcc -o align-demo align-demo.c -m32
[root@localhost c]# ./align-demo
A: 12
```

sizeof(a) + sizeof(b) + sizeof(c) = 1 + 4 + 2 = 7。

A的长度为什么不是7而是12？因为内存对齐。内存对齐需要手工进行吗？我看到，上面的代码并没有手工对齐。而在inode的struct中，我看到了手工对齐，`_unused[16]`。不明白，先不管。

内存对齐怎么让A的长度变成了12？

1. 内存对齐是指，存储数据时，使用的内存空间是`2、4、8`个字节之一。
2. 默认，存储数据，对齐4字节。
3. 在A中，`char a`只需要1个字节，但由于内存空间分配默认是4字节，因此，实际上为a分配了4个字节。
4. `int b`需要4个字节，正好，不需要额外补充。
5. `short c`需要2个字节，不足4个字节，额外补充2个字节。
6. 总计补充的字节：3 + 2 = 5。加上本身需要的字节，7 + 5 = 12。这就是`sizeof(struct A)`的值是12的原因。

我执行上面的代码的操作系统是：

```shell
[root@localhost c]# lsb_release -a
LSB Version:	:core-4.1-amd64:core-4.1-noarch
Distributor ID:	CentOS
Description:	CentOS Linux release 8.2.2004 (Core)
Release:	8.2.2004
Codename:	Core
[root@localhost c]# cat /etc/issue
\S
Kernel \r on an \m
[root@localhost c]# uname -a
Linux localhost.localdomain 4.18.0-193.el8.x86_64 #1 SMP Fri May 8 10:59:10 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
[root@localhost c]# cat /proc/version
Linux version 4.18.0-193.el8.x86_64 (mockbuild@kbuilder.bsys.centos.org) (gcc version 8.3.1 20191121 (Red Hat 8.3.1-5) (GCC)) #1 SMP Fri May 8 10:59:10 UTC 2020
```

内存对齐的规则受操作系统影响。上面的代码在其他操作上执行，结果可能不一致。

内存对齐，有些文章写得很复杂。我看了，没有一眼看懂那些复杂的说明。

了解“内存对齐”，要抓大放小，不阻碍我继续写操作系统，就足够了。

CPU的寄存器有`ax、eax、rax`，分别是2个字节、4个字节、8个字节。和内存交互数据，需要的数据的长度最小是2个字节、4个字节、8个字节。

> 想不下去了。再想下去也没有意义。CPU和内存如何交互数据，有它的规则，我凭空猜想，毫无意义。



## 小结

### 2021-06-24 08:38

做了什么？

1. 中止开发硬盘驱动，转而开发文件系统。没写好文件系统，写好了硬盘驱动，也不好测试。
2. 建立fs存放文件系统代码。更加条理化。
3. 写出v1#在硬盘上制作文件系统#我的设计#概况。

两种方法：

1. 凭记忆，按照书本目录，尽量独立设计和实现文件系统。
   1. 文件系统的很多要素，例如super block，我不理解它存在的必要性。
   2. 因而，我想尽量自己实现文件系统，从而理解文件系统的那些要素存在的必要性。
   3. 学不到东西。
2. 按照书本目录，先看于上神的代码，然后原样仿写出来。
   1. 进度会很快。
   2. 却很类似抄写。
   3. 不易分心。

耗时31分。

### 2021-06-24 09:45

写了v1#在硬盘上制作文件系统#我的设计#流程。耗时56分。

写这样是浪费时间吗？不，这是我在属于我的知识（其实是看过别人的设计之后残存的记忆）基础上的思考。我认为，比直接看于上神的文件系统然后开始实现，更有价值。后者更接近于背诵，而前者，自己思考得更多。

不过，要防止”思而不学则殆“。例如，在”文件操作的并发性“这个知识点，我的选择就是正确的。

虽然是思考，但主要目的是学习。对于一个知识点，我毫无基础，却从零开始思考，这不叫思考，而叫瞎想。费时而低效。我2019年自己动手实现raft和paxos，就是自己瞎折腾。

躺在沙发上写笔记。精神不佳，这个姿势让我效率高一些。

### 2021-06-25 08:54

v1#我的设计#读写硬盘、v1#于上神的设计#数据结构#inode。耗时1个小时9分。

理解不了inode的长度是44字节。我认为，inode的长度应该是512的因素。

自己设计时，及时避免了自己瞎想。做得不错。

常分心，想到一些琐事。休息一下。

### 2021-06-25 08:54

v1#我的设计#读写硬盘、v1#于上神的设计#数据结构#struct和数据对齐。耗时47分。

主要时间消耗在写笔记。

### 2021-06-25 10:27

v1#于上神的设计#数据结构#inode。耗时35分。

分心一分钟左右。新闻的套路。这些都是软新闻，不需要看。

### 2021-06-25 11:01

v1#于上神的设计#数据结构#super block。耗时27分。

看懂了许多，分心比较严重。

一点紧张感都没有。适度紧张一些，效率会高些吧。

就这么一个数据结构，我嫌我看得太慢。不要放纵自己。知识要学，阅读速度也要刻意练习并提高。

### 2021-06-27 09:36

6月26，一整天都没有学习。5点多钟醒了，跑步五公里。半个小时后，很困。太早起床跑步后精神不好，其他时间段例如午休时不午休跑步后精神却很好。

为什么不继续学习？不知道原因。不喜欢学习、不喜欢思考、甚至不喜欢锻炼，是潜伏的本性吧。

非要说原因，大概是：

1. 写文件系统有点无聊。
2. 写文件系统不能一个小时写完，两三个小时也写不完。
3. 就是不想写了。

### 2021-06-27 10:27

v1#于上神的设计#数据结构#super block，理解super block中的`nr_sectors`等成员。耗时42分。

根本不需要耗费这么多时间，最多只需要20分钟就能全部理解完毕。

为什么耗费这么多时间？

1. 间隔了一天，重启，不怎么想看。
2. 分心，想一些琐事。
3. 中途多次散步、喝水、上厕所。

### 2021-06-27 11:04

v1#于上神的设计#数据结构#dir_entry、file_desc。耗时35分钟。

太慢了。原因是：慢不精心，很悠闲。适度紧张一些。

### 2021-06-27 15:47

v1#于上神的设计#我的设计#编码建立文件系统。耗时45分。

我做了什么？

1. 调整本笔记的目录。
2. 写出文件系统的数据结构。

怎么评价这个速度，不紧不慢。有提升空间。

### 2021-06-27 17:21

v1#于上神的设计#我的设计#编码建立文件系统。耗时1个小时01分。

不想写。我想直接看于上神的代码，然后复述。后者，往前推进的速度更快，不容易分心。

常分心。

### 2021-06-29 16:23

前天在知乎上看别的程序员的发展路径，不愿按时睡觉，一直拖延到凌晨3点才睡觉。28日醒来后，不想继续写文件系统。

实现文件系统，枯燥啊，又不能一蹴而就。

若严格要求自己，本可以在7月1日前写完操作系统的。

我说过，人生，存在几个机遇期。

什么叫机遇期？只存在一次的机会。这种机会，机不可失，失不再来。

我若一而再再而三地懈怠，恐怕会丧失某种机遇。

### 2021-06-29 17:01

理解超级快中的`nr_smap_sects`的值；简单回顾被浪费的那些时间。耗时40分。

懒散了两天，再衔接原来的进度，觉得不适应。

### 2021-06-29 17:28

复习一次前天被中断的知识。效率很低，杂念不断。是因为我没有吃午饭吗？

耗时30分。

### 2021-06-29 18:22

看`fs/open.c#do_open`代码。基本理解了。耗时18分。

因琐事而分心。

### 2021-06-30 04:29

看`fs/open.c#search_file`代码，看不懂下面这句

```c
int nr_dir_blks = (dir_inode->i_size + SECTOR_SIZE - 1) / SECTOR_SIZE;
```

凌晨，虽然不困，但很容易分心，效率太低。还是去睡觉吧。

耗时55分。

### 2021-06-30 06:04

看文件系统alloc_smap_bit代码。效果实在太差。分心，想琐事，散步。耗时1个多小时。

凌晨还是好好睡觉吧。

### 2021-06-30 12:50

看文件系统alloc_smap_bit代码。理解不了这句。耗时1个小时17分。

```c
// 1. 为什么减1？
free_sect_nr = (i * SECTOR_SIZE + j) * 8 + k - 1 + sb->n_1st_sect;
```

之前，我以为我理解了这句。翻看那个时候的笔记，推理有误。实际上，并没有理解。

我做了什么？

思维处于无序、无条理的状态，不知道从哪方面去理解问题。只能看这句代码，自由散漫地试图想点什么。

### 2021-06-30 12:50

仍然是上面这个问题。还是没有理解。耗时1个小时57分。

怪不得我不喜欢学习而是喜欢看无用资讯。

看无用资讯，基本上没有看不懂的东西。遇到看不懂的东西，不看就是了。

学习呢？看不懂，还是需要继续看。不知道从哪方面着手才能看得懂。

我做了什么？

1. 分析super block的成员的数值间的关系。
2. 分心。每次遇到理解不了的问题，我就会分心。思维散漫而无序。
3. 看下这里的代码，又看下那里的代码。
4. 盯着的代码看，不知道从哪里入手。

### 2021-06-30 18:18

仍然是上面那个问题。还是没有完全理解为啥要减去1。耗时2个小时。

又看了一些其他代码，想找点线索。可我越看越糊涂了，连其他代码都看出了更多疑问。

不适合再看下去了。纯属浪费时间。

### 2021-07-01 16:26

昨天，受阻于理解不了减去1，越看代码疑问越多，原来没有疑问的地方，也觉得有疑问。

逃不过，避不开，还是需要面对这个问题。

打开github，想把笔记仓库设置成私有仓库，网速太慢。先不设置了。等待时，看了会博客园。

耗时20分钟。

### 2021-07-01 18:15

看`fs/open.c#new_dir_entry`，理解不了。一直看。理解不了，反复看，非常容易分心。中途喝水、散步。太低效了。耗时1小时40分。

### 2021-07-01 22:17

看`fs/open.c#new_dir_entry`，理解了。耗时2个小时30分。有效时间（突然理解了难点的时间）大概五六分钟。

为什么这么慢？

1. 漫不经心。并非故意要这样。理解不了、不知从哪里着手的时候就很容易这样。
2. 散步、喝水、看其他无用资讯。
3. 随便看些代码。

### 2021-07-02 08:00

复习`fs/open.c#new_dir_entry`。用具体例子理解代码的执行过程。

看`fs/main.c#sync_inode`。这其实是我以前透彻理解过的东西，如今再看，又那么陌生。

耗时1个小时10分。

为什么这么慢？思绪散漫，毫无紧张感。

### 2021-07-02 09:00

看文件系统alloc_smap_bit代码。理解不了这句。耗时1个小时6分。

```c
// 1. 为什么减1？
free_sect_nr = (i * SECTOR_SIZE + j) * 8 + k - 1 + sb->n_1st_sect;
```

我做了什么？束手无措，不知道从哪里着手去了解。散步，无力、无条理、无逻辑地在这个问题上耗时间。

先跳过吧。实在没办法了。

### 2021-07-02 13:32

还是上面那个问题。实在低效。毫无头绪地瞎想。耗时1个小时37分。

不甘心跳过去。

### 2021-07-02 17:22

终于理解了下面这句。

```c
// 1. 为什么减1？
free_sect_nr = (i * SECTOR_SIZE + j) * 8 + k - 1 + sb->n_1st_sect;
```

sector-map的第0个bit，和inode-map的第0个bit一样，也是保留位。

这意味着，sector-map的第1个bit，记录数据区的第0个扇区的使用情况。所以，k需要减去1。

百思不得其解，原因是，成见，看出不仔细，以为，sector-map的第0个bit不是保留位。

没有办法，碰运气，再翻翻书。

![image-20210702172803548](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/file-system/image-20210702172803548.png)

耗时44分。有效时间五分钟左右。

想不明白一个问题时，毫无头绪，不知从哪里着手，非常容易分心。

想明白这个问题后，又有点不想继续做了，浪费了这么多时间，还剩余那么多事情要做。

### 2021-07-02 23:00

看了下列函数：

1. new_inode
2. get_inode

不理解：

1. 在get_inode中有可能获取一个非空闲inode。这个inode的i_cnt不为0。
2. 在new_inode中却把i_cnt重置为0。

结合上下文，我知道了：

1. 在new_inode中调用get_inode，一定不会获取一个非空闲inode作为返回值。
2. 在读写文件（非新建）时，直接调用get_inode，此时，才会获取一个非空闲inode作为返回值。

不明白，效率为啥这么低。之前看懂了代码，再看，又好像不懂。

想边看视频边看代码。

耗时1小时37分。

