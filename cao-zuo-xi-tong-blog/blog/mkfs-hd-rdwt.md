# 编码建立文件系统---读写硬盘

## 一些铺垫

选择一个主分区或逻辑分区，把“文件系统”这个数据结构写入到目标分区，就在目标分区建立了文件系统。

我们需要一个读写硬盘的函数才能向目标分区写入数据。这个函数的名称是`hd_rdwt`。`hd_rdwt`属于硬盘驱动。

在本文中，我只梳理主要流程，不叙述`hd_rdwt`使用的其他函数的具体实现过程。对其他函数，我只说明这些函数的用途。

本节中的小标题，每个都是一个函数名。

### hd_cmd_out

向硬盘发送指令，指挥硬盘从`POS`位置开始执行“读取”、“写入”、“识别”硬盘的操作。

### wait_for

使用自旋锁检查硬盘是否准备好了交换数据。

### interrupt_wait

阻塞本进程，一直到硬盘中断发生。

### read_port

函数原型：`void read_port(int port, char *fsbuf, int len)`。

从`port`端口读取`len`字节数据到`fsbuf`地址处。

### write_port

函数原型：`void write_port(int port, char *fsbuf, int len)`。

把`fsbuf`地址处的len字节数据写入到`port`端口。

### 硬盘分区信息

打开硬盘后，我们把硬盘的分区信息存储到了主分区数组`primary`和逻辑分区数组`logical`中。

## 读写硬盘

### 读硬盘

伪代码如下：

```c
void hd_rdwt(Message *msg)
{
  			// 从msg中获取操作硬盘的位置，即，从硬盘的哪个位置开始操作
  			pos = msg->pos;
  			// 从msg中获取次设备号
  			device = msg->device;
  			// 根据次设备号从硬盘分区信息中获取文件系统所在分区的初始物理LBA地址
  			if(device是主分区){
          		lba_base = primary[device];
        }else if(device是次分区){
          		logical_idx = 根据device计算出logical的索引;
          		lba_base = logical[logical_idx];
        }
  			
  			// 计算pos的物理LBA地址sect_nr。
  			// 注意，pos % 512 的结果是0。
  			// 为什么？这是我人为设计的。调用hd_rdwt时，我会保证如此。
  			sect_nr = lba_base + pos >> 9;
  
  			// 向硬盘发送指令，指挥硬盘进行读操作
  			hd_cmd_out(sect_nr)
          
        // 从msg中获取要从硬盘中读取的数据的长度
        len = msg->len;
  			// bytes_left是还要读取的数据的长度
  			bytes_left = len;
  			
  			// 使用一个循环读取数据
  			while(bytes_left){
          	// 先阻塞本进程，一直到硬盘把数据传输到REG_DATA端口
          	interrupt_wait();
          	// 从REG_DATA端口读取bytes字节数据
          	bytes = min(SECTOR_SIZE, len);
          	// 把数据读取到fsbuf地址处，读取bytes字节
          	read_port(REG_DATA, fsbuf, bytes);
          	
          	// 剩余要读的数据的长度
          	bytes_left -= bytes;
          	// 存储数据的新地址
          	fsbuf += bytes;
        }
}
```



### 写硬盘

伪代码如下：

```c
void hd_rdwt(Message *msg)
{
  			// 从msg中获取操作硬盘的位置，即，从硬盘的哪个位置开始操作
  			pos = msg->pos;
  			// 从msg中获取次设备号
  			device = msg->device;
  			// 根据次设备号从硬盘分区信息中获取文件系统所在分区的初始物理LBA地址
  			if(device是主分区){
          		lba_base = primary[device];
        }else if(device是次分区){
          		logical_idx = 根据device计算出logical的索引;
          		lba_base = logical[logical_idx];
        }
  			
  			// 计算pos的物理LBA地址sect_nr。
  			// 注意，pos % 512 的结果是0。
  			// 为什么？这是我人为设计的。调用hd_rdwt时，我会保证如此。
  			sect_nr = lba_base + pos >> 9;
  
  			// 向硬盘发送指令，指挥硬盘进行写操作
  			hd_cmd_out(sect_nr)
          
        // 从msg中获取要写入硬盘的数据的长度
        len = msg->len;
  			// bytes_left是还要写入的数据的长度
  			bytes_left = len;
  			
  			// 使用一个循环写入数据
  			while(bytes_left){
          	// 检查硬盘是否已经准备好了交换数据，如果没有，就空转
          	wait_for();
          	// 向REG_DATA端口写入bytes字节数据
          	bytes = min(SECTOR_SIZE, len);
          	// 从fsbuf地址开始，把bytes字节数据写入到REG_DATA端口
          	write_port(REG_DATA, fsbuf, bytes);
          	// 先阻塞本进程，一直到硬盘把数据从REG_DATA端口取走了bytes字节数据
          	interrupt_wait();
          	
          	// 剩余要写入硬盘的数据的长度
          	bytes_left -= bytes;
          	// 读取数据的新地址
          	fsbuf += bytes;
        }
}
```



### 小结

#### 差异

上面的伪代码高度相似，有两大差异：

1. 调用`hd_cmd_out`。读和写分别向该函数传递不同的参数。
2. `while`中对“延迟”的使用。在下一个小节中叙述。

#### 计算pos的物理LBA地址

调用`hd_cmd_out`读硬盘和写硬盘，都需要向这个函数传递操作硬盘的位置。这里的“位置”必须是物理LBA地址。

什么是“物理LBA地址”？

从msg中获取的pos不是物理LBA地址，而是相对于文件系统所在分区的偏移量。

文件系统所在分区，可以是主分区，也可以是逻辑分区。

在主分区内，某个扇区的偏移量是OFF，这个扇区的物理LBA地址 = 主分区的初始物理LBA地址 + OFF。

在逻辑分区内，某个扇区的偏移量是OFF，这个扇区的物理LBA地址 = 逻辑分区的初始物理LBA地址 + OFF。

逻辑分区的初始物理LBA地址，可以从硬盘分区信息的逻辑分区logical中获取。

在逻辑分区内，某个扇区的偏移量是OFF，这个扇区的物理LBA地址的计算公式实际上是：
$$
扇区的物理LBA地址 = 逻辑分区所在主分区的初始物理LBA地址 + 逻辑分区相对它所在的主分区的偏移量 + OFF。
$$
由于我们在硬盘分区信息中存储的是每个分区相对于它所在的主分区的绝对物理地址，所以，上面的公式变成了：
$$
扇区的物理LBA地址 = 逻辑分区的初始物理LBA地址 + OFF。
$$

## 延迟

操作硬盘的流程大致如下：

1. 通过`hd_cmd_out`指挥硬盘进行操作。
2. 硬盘接收指令后，进行下面两种操作中的一种：
   1. 把数据从硬盘中传输到`REG_DATA`端口。这是读硬盘。
   2. 把数据从`REG_DATA`端口取走写入硬盘。这是写硬盘。
3. 硬盘驱动和`REG_DATA`端口交互。
   1. 写硬盘：把数据写入`REG_DATA`端口。
      1. 向`REG_DATA`端口写入数据前，要确保硬盘已经准备好传输数据，因此需要使用`wait_for`。
      2. 把数据写入`REG_DATA`后，要让硬盘把这个端口的数据取走后，才能再次写数据，否则，可能导致数据在端口堆积。
   2. 读硬盘：从`REG_DATA`端口读数据。硬盘把数据传输到`REG_DATA`端口后，硬盘驱动才能去读取数据。否则，硬盘驱动读取不到数据。

我其实想在叙述上面的流程时讲讲硬盘中断，但没有找到合适的切入点。硬盘中断实在很重要。

键盘中断是怎么发生的？当我们按下或松开按键的时候。

同样的道理，硬盘中断也有一个触发点。当硬盘和`REG_DATA`端口传输数据完毕时，就会发生一次硬盘中断。

`wait_for`通过自旋锁阻塞硬盘驱动。

`interrupt_wait`通过IPC机制阻塞硬盘驱动。当硬盘中断发生时，硬盘中断例程会解除硬盘驱动的阻塞。

## 小结

### 2021-07-05 23:16

这篇文章写得还行。伪代码的表现力很强。如果用文字，不容易把`hd_rdwt`讲得这么清晰。

对`hd_rdwt`掌握到这种程度，就不需要再纠结了，明天就动手写代码。

耗时1小时22分。

### 2021-07-05 23:16

排版、发布、查看。耗时10分。

