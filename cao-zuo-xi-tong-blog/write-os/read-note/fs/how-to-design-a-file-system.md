# 怎么设计一个文件系统

## 文件系统是什么

文件系统是一种数据结构。利用这种数据结构，操作系统能够在硬盘中创建一个文件或找到一个文件，然后从文件中读取数据，或者向文件中写入数据。当然，操作系统也可以删除文件。

## 一个简单的文件系统

本文要介绍的文件系统和Linux的文件系统类似，但是和Linux相比，简化了许多。

我们的文件系统是一种“扁平”的文件系统。所谓“扁平”，是指只支持一级目录。

文件系统包括两部分，第一部分是数据结构，我认为这是狭义上的文件系统；第二部分是文件系统中包含的数据，也就是我们在文件系统中存储的数据。

文件系统包含哪些数据结构呢？下面的小标题中，除了“引导扇区”和“数据区”，都是构成文件系统的数据结构。

### 引导扇区

我没有弄清楚这是什么。在编码建立文件系统时，只需预留一个扇区作为引导扇区即可。

### 超级块

英文术语是“super block"。

它存储文件系统的元数据。代码如下：

```c
struct super block{
  	// 文件系统的魔数。
  	int magic_number;
  	// 有多少个inode。
  	int nr_inodes;
  	// inode-map占用多少个扇区。
  	int nr_inode_map_sectors;
  	// sector-map占用多少个扇区。
  	int nr_sector_map_sectors;
  	// 一个inode占用多少字节。
  	int inode_size;
  	// 数据区域的第一个扇区的LBA地址
  	int 1st_data_sector;
  	// 安装文件系统的分区总计有多少个扇区
  	int nr_sectors;
  	// 指向根目录的inode在inode-map中的索引。
  	int root_inode;
  	// 一个目录项的大小。
  	int dir_entry;
  
  	// 安装文件系统的设备的设备号。只存在于内存中。
  	int dev;
};
```

什么叫魔数？

可以理解为用一个标志代表另一个事物。

例如，有A、B、C三种文件系统。分别用`0x0`、`0x1`、`0x2`表示这三种文件系统。

假如，我们在文件系统的超级块中发现魔数是`0x0`，就可以认为，这种文件系统是A。魔数和文件系统的种类互为充要条件。

### inode-map

我们的inode-map只有一个扇区。这个扇区的每个bit记录一个inode-array中的一个inode的使用情况。

补充一点，inode-map的第0个bit不记录inode-array中的inode的使用情况，作为保留位使用。

为啥作为保留位？我暂时没有弄明白。读者若有兴趣，在编码实现文件系统的过程中，可以试试不把第0个bit作为保留位。

inode-map中的第1个bit记录inode-array中的第0个元素的使用情况。

inode-map中的第2个bit记录inode-array中的第1个元素的使用情况。

inode-map中的第3个bit记录inode-array中的第2个元素的使用情况。

inode-map中的第N个bit记录inode-array中的第N-1个元素的使用情况。

inode-map中的第N个bit的值是0，表示inode-array中的第N-1个元素没有指向任何文件。

inode-map中的第N个bit的值是1，表示inode-array中的第N-1个元素指向了一个文件。

### inode-array

```c
struct inode{
  	// 文件类型。
  	int i_mode;
  	// 文件大小。
  	int i_size;
  	// 文件开始的扇区
  	int i_start_sector;
  	// 一个文件占用的扇区数量
  	int i_nr_sectors;
  	char _unused[16];
  
  	// 只存在于内存中的成员
  	// 设备号
  	int i_dev;
  	// 有多少个进程共享这个inode，通俗说法，有多少个进程打开了这个inode指向的文件。
  	int i_cnt;
  	// 这个inode对应inode-map中的第多少个bit
  	int i_num;
};
```

`i_mode`是指这个inode指向的文件是普通文件还是终端文件。

一个inode也能指向终端文件吗？

在Linux中，”一切皆文件“。我们的文件系统，也要体现这个思想。我们的文件系统，实际是简化版的虚拟文件系统，在将来，会把终端也接入文件系统中。对终端的读写，也通过这个文件系统。

`char _unused[16];`，没有实际意义，只是为了内存对齐。这涉及到”struct和内存对齐“的知识。本代码基于32位CPU。在32位CPU中，一个数据的大小如果是32的倍数，能加快CPU读写内存。

### sector-map

这个数据结构和inode-map类似，用bit记录数据区的扇区的使用情况。

sector-map的第0个bit也用来记录扇区的使用情况。

### 根目录

先介绍一个数据结构。代码如下：

```c
struct dir_entry{
  		// inode-map中的索引。
  		int inode_index;
  		// 文件名
  		char name[20];
};
```

每个`dir_entry`指向一个文件。

在文件系统中存在多少个文件，在根目录中就有多少个对应的`dir_entry`。

在文件系统中查找文件的流程如下：

1. 目标文件的文件名是"filename"。
2. 遍历根目录，检查每个`dir_entry`的文件名是否等于"filename"。
   1. 否，继续查找下一个`dir_entry`。
   2. 是，停止查找，当前`dir_entry`就是目标`dir_entry`。
3. 根据目标`dir_entry`中的`inode_index`在inode-array中找到目标inode。目标inode是`inode-array[inode_index-1]`。
4. 从目标inode中获取目标文件的元数据，找到文件。

### 数据区

没啥好解释的，就是存储文件的具体数据的区域。

## 参考资料

《一个操作系统的实现》