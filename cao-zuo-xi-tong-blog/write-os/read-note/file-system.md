# 文件系统

本文是我阅读《一个操作系统的实现》后的复述，不保证正确性。

## 硬盘简介

### v1

硬盘有新旧两个标准。

很久以前，硬盘控制器和硬盘本身是分离的。

IDE0、IDE1。它们位于主板上。

硬盘有两类寄存器：块控制器和命令控制器。

怎么读写硬盘？

往块控制器写入数据，往命令控制器写入命令。

命令控制器，接收指令做什么，例如，“读硬盘“。

块控制器，接收数据，例如，从硬盘的什么位置读数据。

### v2

写硬盘驱动，用到硬盘的两组寄存器：命令寄存器、控制块寄存器。

两者的作用分别是什么？一组很多，一组很少。

有一组寄存器，同一个寄存器，却有两个不同的名称。读硬盘时，A寄存器叫小A；写硬盘时，A寄存器叫小b。小A和小b，如何体现它们是同一个寄存器呢？操作小A和小B是，都是和同一个端口交互数据。

LBA和CHS。这是两种读写存储工具的方式。CHS，在写loader读取软盘时，我已经用过了，需要计算一番。对人不友好，对机器却很友好，因为，存储设备的结构就是那样的，柱面---》磁头---》扇区。

LBA是对人很友好的寻址方式。对，准确的称呼，应该是寻址方式。

我要写的硬盘驱动程序，使用的寻址方式是LBA28。"28"是什么意思？它表示，用28个bit表示扇区编号，能表示的最大数是
$$
2^{28}-1
$$
把0也计算在内，能表示的扇区数量是
$$
2^{28}
$$
能表示的硬盘总容量是
$$
512 * 2^{28}
$$
28个bit，分布在device和?寄存器的High、Mid、Low。

8259A。硬盘中断，用8259A的从片的端口的第14个bit控制；8259A的从片又必须级联到主片的第2个bit位。

> 于上神的书写得过于简略，要搭配郑上神的书看才行。
>
> 看了郑上神的硬盘章节，我才明白于上神在说什么。

### v3

#### Device Register

一共八位，即，一个字节。

1. ~~0~3，L-->1，是LBA28的最后4位。L->0，是磁头号。~~
2. 如果L-->1，0~3是LBA28的最后4位。如果L->0，是磁头号。
3. ~~第4位，drv，1-->slave，0-->master~~
4. 第4位是drv。如果drv是1，读写的硬盘是slave。如果drv是0，读写的硬盘是master。
5. 第6位，L，1--》LBA，0-->CHS
6. 第5位、第7位，固定是1。不理会。

> 不用宏，用struct，会很容易理解。

> 笔记写得不全。L是什么？

![image-20210525090935350](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/image-20210525090935350.png)

### v4

查看到的硬盘参数，怎么理解？在下面的图片，有详细说明。

全部资料在下面的链接。

https://web.archive.org/web/20040327130655/http://www.t13.org/project/d1410r2a.pdf

关于ATA的所有文档链接在

https://web.archive.org/web/20040403173111/http://t13.org/

![image-20210516220010575](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/image-20210516220010575.png)

图中第一列的数字，单位是字。

第10个字到第19个字，共10个字，存储的是Serial number。

第27个到第46个字，共20个字，存储的是Model number。

```c
// kernel/hd.c
struct iden_info_ascii {
                int idx;
                int len;
                char * desc;
        } iinfo[] = {{10, 20, "HD SN"}, /* Serial number in ASCII */
                     {27, 40, "HD Model"} /* Model number in ASCII */ };

        for (k = 0; k < sizeof(iinfo)/sizeof(iinfo[0]); k++) {
                char * p = (char*)&hdinfo[iinfo[k].idx];
                for (i = 0; i < iinfo[k].len/2; i++) {
                        s[i*2+1] = *p++;
                        s[i*2] = *p++;
                }
                s[i*2] = 0;
                printl("%s: %s\n", iinfo[k].desc, s);
        }
```

`hdinfo[iinfo[k].idx`的值依次是：`10`、`27`。

为什么要这样？

```shell
char * p = (char*)&hdinfo[iinfo[k].idx];
for (i = 0; i < iinfo[k].len/2; i++) {
  s[i*2+1] = *p++;
  s[i*2] = *p++;
}
s[i*2] = 0;
```

从硬盘中读取到的硬盘数据的单位是字，16个bit，2个字节，大端法。

iinfo[k].len的长度的单位是字节，所以要除以2。

> 和大端法、小端法无关，也和是否是字没有必然关系。
>
> Serial number和Model number的字符串是颠倒的。用例子说明更准确。
>
> 例如，"World!"，存储在硬盘中的数据却是，“oWlr!d”。
>
> 我们的任务就是，把硬盘中的那种字符串还原成"World!"。
>
> 这种字符串，每两个字符之间的顺序是正确的，两个字符之间的顺序是颠倒的。每次处理一对，颠倒这一对字符。

~~内存和CPU中的数据，小端法（高地址数据放在左边，低地址数据放在右边）。~~

我不知道从硬盘中读取的数据和内存中的数据的在大小端方面是否一致。从于上神的代码看，二者的顺序是颠倒的。

![image-20210516232337894](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/image-20210516232337894.png)



​	![image-20210516232546275](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/image-20210516232546275.png)



| 0    | 1     |      |      |      |      |      |      |
| ---- | ----- | ---- | ---- | ---- | ---- | ---- | ---- |
| i*2  | i*2+1 |      |      |      |      |      |      |
| B    | A     |      |      |      |      |      |      |
|      |       |      |      |      |      |      |      |

| 0    | 1    |      |      |      |      |      |      |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| *p++ | *p++ |      |      |      |      |      |      |
| A    | B    |      |      |      |      |      |      |
|      |      |      |      |      |      |      |      |

小端法存储`0x124`

把数值的低位存储到内存的低位。

| 内存地址 | 0    | 1    | 2    |      |      |      |      |
| -------- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 数据     | 4    | 2    | 1    |      |      |      |      |
|          |      |      |      |      |      |      |      |

大端法存储`0x124`

把数值的高位存储到内存的低位。

| 内存地址 | 0    | 1    | 2    |      |      |      |      |
| -------- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 数据     | 1    | 2    | 4    |      |      |      |      |
|          |      |      |      |      |      |      |      |

![在这里插入图片描述](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/20200313201318115.png)



outsw

https://www.fermimn.edu.it/linux/quarta/x86/outs.htm

### v5

主设备号，让操作系统识别是硬盘还是软盘还是其他，作用是根据不同种类的硬件选择不同的驱动程序。

次设备号，让操作系统知道是哪个硬件，例如哪块硬盘的哪个区。

### v6

每个扩展分区最多有16个逻辑分区。为什么？

并不是这样的。书中的这句话应该这样理解：

1. 一个硬盘，最多能划分为四个主分区，最多只有一个扩展分区，每个主分区都可以作为扩展分区，但是，扩展分区只有一个。
2. 每个扩展分区能划分出若干个逻辑分区，并不是最多只有16个逻辑分区。
3. 最多有16个逻辑分区，只是于上神设计硬盘分区方案时的一种规定。

分区表的原理：

1. 只有一个MBR。MBR的结构是：
   1. MBR占用一个扇区，最后两个字节是魔数`55AA`，在它前面的64个字节是分区元数据，剩余的(512-66=446)个字节是引导扇区。
   2. 每个分区的元数据占用16个字节，64个字节能表示四个分区。
   3. 这四个分区能全部用来表示主分区，也能用来表示3个主分区和一个扩展分区。记住，最多只能有一个扩展分区。
2. 可以有多个EBR。EBR的结构和MBR相似，差异是，EBR的分区部分只表示两个分区
   1. 一个是当前逻辑分区
   2. 另一个是下一个扩展分区
   3. 这是一个链表。

> 2021-06-16 15:37 文件系统是我一个月前看过的知识。沮丧。分区表，我又理解得不透彻了。

### v7

设备号。主次设备号。主设备号，识别哪个驱动程序；次设备号，识别哪个硬盘的哪个区。

### v8

![image-20210517225314438](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/image-20210517225314438.png)

```shell
[root@localhost b]# fdisk -l ./80m.img
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
```

![image-20210517225449994](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/image-20210517225449994.png)

80m.img5的命名为什么不是hd3a？

因为，第一块硬盘的第一个分区的名称是hd1，第一块硬盘的第二个分区的名称是hd2。

第一块硬盘的第二个分区是80m.img2。80m.img5是第80m.img2的逻辑分区。每个分区可以拥有若干个逻辑分区（每个分区，可以理解成一块虚拟硬盘，这个虚拟硬盘能分成若干个逻辑分区，这是和真实硬盘的差异）。我们规定，每个分区只有16个逻辑分区。

如果一个分区的名称是`hd2`，那么，这个分区的逻辑分区的名称分别是：`hd2a、hd2b、hd2c、hd2d、hd2e、hd2f`等。

> 再次证明，笔记，务必要尽量写得详细再详细。上面这段话，我看了很久才写出来。在八天前，我认为，这句话不必写出来，能理所当然地想出来。



> 上图中的小格子如`hd1a`等，都是逻辑分区。
>
> 2021-06-17 14:53 补充。

### v9

理解了下面这个宏。

```c
// NR:Number
// MAX_PRIM 9
// NR_PRIM_PER_DRIVE 5
// MINOR_hd1a 0x10
// NR_SUB_PER_DRIVE 64
#define DRV_OF_DEV(dev) (dev <= MAX_PRIM ? \
                         dev / NR_PRIM_PER_DRIVE : \
                         (dev - MINOR_hd1a) / NR_SUB_PER_DRIVE)
```

device 的寄存器的第 4位，指定通道上的主或从硬盘，0为主盘， 1为从盘。

`dev - MINOR_hd1a`，为何要减去16？图中的x是16，在于上神的设计中。当然，x也可以是0。如果x不是0，即使是1，也会导致x+127是64的两倍，驱动器号是2，这是错误的。

上面的这句话，有问题。更正成下面这段话。

x可以是任意值。如果x是1，x+127-x，结果是127，除以64，结果仍然是1。

宏的逻辑是：

1. 如果设备号小于等于9，那么，这个设备是主分区。
2. 如果设备号大于9，那么，这个设备是扩展分区的逻辑分区。
3. 再区分是master硬盘还是slave硬盘的分区。
   1. 如果是主分区，判断方法是：用设备号除以5。
      1. 商是0，是master硬盘。
      2. 商是1，是slave硬盘。
   2. 如果是逻辑分区，判断方法是：用设备号除以64。
      1. 商是0，是master硬盘。
      2. 商是1，是slave硬盘。

这个宏的作用是什么？根据分区号计算出设备号，即第几块硬盘。

> 2021-06-16 15:51 看不懂上面的文字。

### v10

```c
// file:/home/cg/yuyuan-os/osfs10/e/kernel/hd.c

PRIVATE u8              hdbuf[SECTOR_SIZE * 2];

PRIVATE void hd_identify(int drive)
{
        struct hd_cmd cmd;
        cmd.device  = MAKE_DEVICE_REG(0, drive, 0);
        cmd.command = ATA_IDENTIFY;
        hd_cmd_out(&cmd);
  			// 不理解这个中断
        interrupt_wait();
        port_read(REG_DATA, hdbuf, SECTOR_SIZE);

        print_identify_info((u16*)hdbuf);

        u16* hdinfo = (u16*)hdbuf;

        hd_info[drive].primary[0].base = 0;
        /* Total Nr of User Addressable Sectors */
        // 小端法
        hd_info[drive].primary[0].size = ((int)hdinfo[61] << 16) + hdinfo[60];
}
```

> 2021-06-16 16:06 一定要写上代码的文件名。我想在项目中找到代码，却不知道它出自哪个文件。

### v11

```c
// file：/home/cg/yuyuan-os/osfs10/e/kernel/hd.c
/*****************************************************************************
 *                                partition
 *****************************************************************************/
/**
 * <Ring 1> This routine is called when a device is opened. It reads the
 * partition table(s) and fills the hd_info struct.
 * 
 * @param device Device nr.
 * @param style  P_PRIMARY or P_EXTENDED.
 *****************************************************************************/
PRIVATE void partition(int device, int style)
{
	int i;
  // 根据设备号ddevice计算出使用哪个硬盘。
	int drive = DRV_OF_DEV(device);
	struct hd_info * hdi = &hd_info[drive];

	struct part_ent part_tbl[NR_SUB_PER_DRIVE];

	if (style == P_PRIMARY) {
		get_part_table(drive, drive, part_tbl);

		int nr_prim_parts = 0;
		for (i = 0; i < NR_PART_PER_DRIVE; i++) { /* 0~3 */
			if (part_tbl[i].sys_id == NO_PART) 
				continue;

			nr_prim_parts++;
			int dev_nr = i + 1;		  /* 1~4 */
      // 1. 这是规定。分区表中的表项的结构就是这样的。表项记录了一个分区的开始扇区、拥有的扇区的数量。
      // 2. 开始扇区，是什么？是绝对LBA还是其他？
			hdi->primary[dev_nr].base = part_tbl[i].start_sect;
			hdi->primary[dev_nr].size = part_tbl[i].nr_sects;

			if (part_tbl[i].sys_id == EXT_PART) /* extended */
				partition(device + dev_nr, P_EXTENDED);
		}
		assert(nr_prim_parts != 0);
	}
	else if (style == P_EXTENDED) {
		int j = device % NR_PRIM_PER_DRIVE; /* 1~4 */
		int ext_start_sect = hdi->primary[j].base;
		int s = ext_start_sect;
		int nr_1st_sub = (j - 1) * NR_SUB_PER_PART; /* 0/16/32/48 */

		for (i = 0; i < NR_SUB_PER_PART; i++) {
			int dev_nr = nr_1st_sub + i;/* 0~15/16~31/32~47/48~63 */

			get_part_table(drive, s, part_tbl);

			hdi->logical[dev_nr].base = s + part_tbl[0].start_sect;
			hdi->logical[dev_nr].size = part_tbl[0].nr_sects;

			s = ext_start_sect + part_tbl[1].start_sect;

			/* no more logical partitions
			   in this extended partition */
			if (part_tbl[1].sys_id == NO_PART)
				break;
		}
	}
	else {
		assert(0);
	}
}
```



> 2021-06-16 16:27 看不懂上面的代码。一个月前，我是看懂了的。
>
> 2021-06-17 12:11 仍然看不懂上面的代码。我想，关键是要弄清楚分区的次设备号。 



一、`partition(device + dev_nr, P_EXTENDED);`，修改成`partition(dev_nr, P_EXTENDED);`行不行？

当dev_nr等于1时，`partition(dev_nr, P_EXTENDED);`是`partition(1, P_EXTENDED);`。

`int drive = DRV_OF_DEV(1);`执行后，drive是多少？是0。

用`partition(device + dev_nr, P_EXTENDED);`计算。

1. `partition(device + dev_nr, P_EXTENDED);`
2. `partition(0 + dev_nr, P_EXTENDED);`
3. 结果和`partition(dev_nr, P_EXTENDED);`相同。是否可以认为，`partition(device + dev_nr, P_EXTENDED);`，能修改成`partition(dev_nr, P_EXTENDED);`？

综合全局考虑后，回答是不行。理由如下：

在`hd_open`中，`partition(drive * (NR_PART_PER_DRIVE + 1), P_PRIMARY);`，NR_PART_PER_DRIVE是4。

> 在`hd_open`中，`partition(drive * (NR_PART_PER_DRIVE + 1), P_PRIMARY);`，NR_PART_PER_DRIVE是4。
>
> 这句话没头没尾，很不好理解。
>
> ```c
> PRIVATE void hd_open(int device)
> {
>         int drive = DRV_OF_DEV(device);
>         assert(drive == 0);     /* only one drive */
> 
>         hd_identify(drive);
> 
>         if (hd_info[drive].open_cnt++ == 0) {
>                 partition(drive * (NR_PART_PER_DRIVE + 1), P_PRIMARY);
>                 print_hdinfo(&hd_info[drive]);
>         }
> }
> ```
>
> 1. `drive * (NR_PART_PER_DRIVE + 1)`的结果是0或5。
> 2. 到了partion函数中的`partition(device + dev_nr, P_EXTENDED);`这句，`device`是0或5。
> 3. 在第2步中的partitiion中，device在[0, 4]或[5, 9]这两个区间。
> 4. 然后，在partition的DRV_OF_DEV中，计算结果一定是0或1。能识别出是哪块硬盘。
> 5. 为啥理解起来这么费劲？因为原来记录的笔记省略了上下文。
>
> 2021-06-17 14:27 补充。

1. 当drive = 0时，`partition(drive * (NR_PART_PER_DRIVE + 1), P_PRIMARY);` 是 `partition(0, P_PRIMARY);`。
   1. 在`partition(0, P_PRIMARY);`中，
      1. `int drive = DRV_OF_DEV(device);`执行后，drive是0。
      2. `struct hd_info * hdi = &hd_info[drive];`执行后，hdi是第一块硬盘。
      3. `partition(device + dev_nr, P_EXTENDED);`是`partition(0 + dev_nr, P_EXTENDED);`
         1. dev_nr的值是`1~~4`。
         2. `partition(0 + dev_nr, P_EXTENDED);`中，`int drive = DRV_OF_DEV(0+dev_nr);`的值总是0。
         3. 因为，0+dev_nr不大于4，4/5 = 0。
2. 当drive = 1时，`partition(drive * (NR_PART_PER_DRIVE + 1), P_PRIMARY);` 是 `partition(5, P_PRIMARY);`。
   1. 在`partition(5, P_PRIMARY);`中，
      1. `int drive = DRV_OF_DEV(5);`执行后，drive是1。为什么？看看宏DRV_OF_DEV就知道了。
      2. `struct hd_info * hdi = &hd_info[1];`执行后，hdi是第二块硬盘。
      3. `partition(device + dev_nr, P_EXTENDED);`是`partition(5 + dev_nr, P_EXTENDED);`
         1. dev_nr的值是`1~~4`。partition的第一个参数总是大于0。
         2. `partition(5 + dev_nr, P_EXTENDED);`中，`int drive = DRV_OF_DEV(0+dev_nr);`的值总是1。

> 重看上面的思考记录，没有一眼看明白。
>
> 把`partition(device + dev_nr, P_EXTENDED);`，能修改成`partition(dev_nr, P_EXTENDED);`后，无法根据次设备号识别出是第几块硬盘。因此，不能修改。
>
> 2021-06-17 14:27 补充。



> 想明白了。
>
> 每个分区（包括硬盘）都拥有一个独一无二的次设备号。
>
> 一个次设备号只能表示一个分区（包括硬盘），不能用一个次设备号既表示分区又表示硬盘。
>
> 两块硬盘，主分区，次设备号分别是：0、1、2、3、4、5、6、7、8、9。
>
> 扩展分区没有次设备号。
>
> 扩展分区被继续分割，存在于扩展分区中的逻辑分区才有分区号。
>
> 在前面，我说过，次设备号不能重复。因此，逻辑分区的次设备号应该大于等于10。如果能给等于9，那么，次设备号9指向哪个设备？这再次证明，次设备号和设备（分区）是一一对应的。
>
> 之前理解不了分区号，但却没意识到我理解不了的是这个知识点。逐渐确认我的疑惑点后，一举知道了这个问题的答案。找出问题比找出答案更费时。
>
> 2021-06-17 14:58 补充。

硬盘最开始的扇区，由三部分组成：

1. 主引导记录：MBR。
2. 磁盘分区表：DPT。
3. 魔数：55AA。

每个EBR中的start_sector的基址都是作为扩展分区的主分区的base地址吗？这是规定，找到资料验证一下就知道了。

子扩展分区是在总扩展分区中创建的，子扩展分区的偏移扇区理应以总扩展分区的绝对扇区 LBA 地址为基准，因此，＂子扩展分区的绝对扇区 LBA 地址＝总扩展分区绝对扇区 LBA 地址＋子扩展分区的偏移扇区”。逻辑分区是在子扩展分区中创建的，逻辑分区的偏移扇区理应以子扩展分区的绝对扇区 LBA 地址为基准，因此，“逻辑分区的绝对扇区 LBA 地址＝子扩展分区绝对扇区 LBA 地址＋逻辑分区偏移扇区“这里的子扩展分区就是当前子扩展分区。

> 2021-06-16 16:29 呵呵。我看不懂自己一个月前写的这段话。

### v12



```c
/*****************************************************************************
 *                                get_inode
 *****************************************************************************/
/**
 * <Ring 1> Get the inode ptr of given inode nr. A cache -- inode_table[] -- is
 * maintained to make things faster. If the inode requested is already there,
 * just return it. Otherwise the inode will be read from the disk.
 * 
 * @param dev Device nr.
 * @param num I-node nr.
 * 
 * @return The inode ptr requested.
 *****************************************************************************/
PUBLIC struct inode * get_inode(int dev, int num)
{
	if (num == 0)
		return 0;

	struct inode * p;
	struct inode * q = 0;
  // 这种遍历数组的方法，我曾经非常熟悉。现在，又感觉有点陌生了。
	for (p = &inode_table[0]; p < &inode_table[NR_INODE]; p++) {
		if (p->i_cnt) {	/* not a free slot */
			if ((p->i_dev == dev) && (p->i_num == num)) {
				/* this is the inode we want */
				p->i_cnt++;
				return p;
			}
      // 我认为，这里应该加上continue
      // 代入具体数值计算，才能验证是否正确。
      continue;
		}
		else {		/* a free slot */
			if (!q) /* q hasn't been assigned yet */
				q = p; /* q <- the 1st free slot */
		}
	}

	if (!q)
		panic("the inode table is full");

  // 实际上是对inode_table中的对应的元素的修改。
  // 要注意这里。对指针不熟悉，就会非常困惑。
  // 使用指针指向inode缓存inode_table的作用，不必显示把更新后的inode保存到inode_table，而是直接读写inode_table中的数据。
	q->i_dev = dev;
	q->i_num = num;
	q->i_cnt = 1;

	struct super_block * sb = get_super_block(dev);
	int blk_nr = 1 + 1 + sb->nr_imap_sects + sb->nr_smap_sects +
		((num - 1) / (SECTOR_SIZE / INODE_SIZE));
	RD_SECT(dev, blk_nr);
	// 作用是什么？
	// 这种给struct赋值的方式，没有问题。之前已经知道了，struct就是一段内存中的数据。
	// 我不理解的是，fsbuf中偏移几个inode后就是新inode的数据。
	// 1. 在fsbuf中，inode是连续的吗？
	// 2. 在fsbuf中，只有inode吗？
	// 3. fsbuf和inode_table有没有关系？
	struct inode * pinode =
		(struct inode*)((u8*)fsbuf +
				((num - 1 ) % (SECTOR_SIZE / INODE_SIZE))
				 * INODE_SIZE);
	// 仅仅是初始化吗？
  // 找到这个free的inode，目的就是重新设置它的值。它原来的值是什么，在这个函数中，我不关心，它原来的值没有任何用处。
	q->i_mode = pinode->i_mode;
	q->i_size = pinode->i_size;
	q->i_start_sect = pinode->i_start_sect;
	q->i_nr_sects = pinode->i_nr_sects;
	return q;
}
```



`int blk_nr = 1 + 1 + sb->nr_imap_sects + sb->nr_smap_sects + ((num - 1) / (SECTOR_SIZE / INODE_SIZE));`中的`((num - 1) / (SECTOR_SIZE / INODE_SIZE))`是什么意思？

意思是：num对应的inode在inode-array中的位置在哪个扇区(其实就是把num换算成inode-array区域的扇区号）。(2021-06-16 16:48 补充)	

### v13

实在是烦人。之前理解了知识点，我现在又觉得有问题。这样下去，什么时候能写完操作系统？烦死了！

用LBA的方式操作硬盘。LBA的初始值是0。

如果给硬盘的扇区编号，编号的初始值是1，那么，

要读取1号扇区，LBA = 0。

要读取2号扇区，LBA = 1。

要读取N号扇区，LBA = N-1。

这是没有问题的。

安装了文件系统的分区，在inode-array区域前，有A个扇区。

要读取inode-array的第一号扇区(A+1)，LBA = A。

有没有问题？没有问题。

使用RW_SECT把第(A+1)号扇区读取到fsbuf中，在计算fsbuf + X时，fsbuf的初始值是多少？

fsbuf的初始值是第(A+1)号扇区的开始位置，而不是第(A+1)号扇区的结束位置。这个认识，太太重要了！

在get_inode中，RW_SECT读取的这个扇区有什么特点？这个扇区存储一个未被使用的inode结构。这句话，也非常重要。

因为，我担忧：对这个未被使用的inode结构设置值会擦除它原来的值。

呵呵，杞人忧天！我为什么很容易担忧那些不应该担心的问题呢？

为什么这样说？找到这个free的inode，目的就是重新设置它的值。它原来的值是什么，在这个函数中，我不关心，它原来的值没有任何用处。

### v14

#### 我的思路

理解不了`do_rdwt`。

毫无头绪。试试，如果我来实现读写功能，会怎么做？

在文件描述符中，记录了操作文件时的初始位置：pos。

简化问题，先分析读文件。

从硬盘中读取文件数据到内存中，使用RW_SECT。

RW_SECT需要几个参数？

1. 从第几号扇区开始读？
2. 读多少个扇区？
   1. 最终要读取的数据的长度是L字节。L是读文件函数的第二个参数。
   2. RW_SECT读取数据的单位是扇区，每次读取1个扇区或2个扇区或3个扇区或N个扇区，假设RW_SECT每次读取的扇区数量是N。
   3. RW_SECT每次读取的数据都是N个扇区，不受L影响。
   4. L影响的是phycopy的参数。这个函数把从硬盘中读取的数据放到存储文件数据的buf中。
   5. phpycopy每次复制多少数据？复制min(L, N个扇区)。
   6. 可以用一个很简陋的方法，把N设置得非常大，然后，每次L都是更小的，只需用RW_SECT读取一次，从fsbuf中复制L数据到buf中就可以了。
   7. 使用这种方法，技术含量太低了。
   8. 多次使用RW_SECT，需要使用几次？
   9. 设置一个变量，bytes_left，剩余要读的数据的长度。使用一个循环。
      1. 循环运行的条件是bytes_left>0。
      2. 使用RW_SECT
         1. 两个参数：初始扇区是第几号扇区S，读取多少个扇区N
         2. N是固定值。
         3. S会发生变化。每次增加N。
      3. phycopy复制多少个字节数据？
         1. 比较N和bytes_left，copy_bytes = min(N, bytes_left)是目标数据。
      4. bytes_left = bytes_left - copy_bytes。
   10. bytes_left方法，读取数据没有问题。读完数据之后，设置文件的pos，pos = len。

> 自己不会就学习别人的方法，然后记在心中。
> 2021-6-16 20:06 看不懂我以前写的笔记。

### v15

删除文件，link.c。

#### 我的思考

删除文件，只需把文件对应的inode、sector标记成未被使用，也就是设置inode-map和sector-map。

怎么根据需要设置inode-map、sector-map中的对应bit呢？对我来说，不是一个很简单的问题。

在文件描述符中记录了inode在inode-array中的索引A。

在inode中记录了文件的第一个扇区的扇区号F、这个文件的大小L、这个文件占用多少个扇区M。

更新inode-map和sector-map，需要读写硬盘。

##### inode-map

A和inode-map的索引in的关系是：A = in - 1，A >= 0。

in = A + 1。

sector_no = in / a_sector_size。

偏移量off = in % a_sector_size。

把包含我要修改的那部分inode-map中的数据读入内存，读取单位是扇区。

读取扇区，需要确定开始扇区，读取大小设置成一个扇区。只需要设置成一个扇区。因为，我的目标数据只是一个bit。

疑点出来了。开始扇区如何确定？

这个疑点，还是那个常见的基本逻辑：计算机中的索引，初始值是0，计算第N个索引所表示的数据。

start_sector_no = 1 + 1 + sector_no。读取数据时，初始扇区号是start_sector_no，读取的是inode-map区域的第一个扇区。

把目标扇区存储到fsbuf中。这个扇区的第off（初始值是0）就是目标bit。

不使用循环遍历，这个方法的效率太低，时间复杂度是O(N)。

我的方法是：

1. off的单位是bit。
2. byte_index = off / 8。这是包含目标bit的字节在fsbuf中的偏移量，单位是字节。
3. bit_off = off % 8。这是目标bit在字节中的偏移量。
4. fsbuf[byte_index]是包含目标bit的那个字节。这个字节的第bit_off应该设置成0。
5. fsbuf[byte_index] = fsbuf[byte_index] & (~1<bit_off).

> 2021-06-16 20:09 完全不清楚我上面写的是什么？那些是我一个多月前写的。

##### sector-map

#### 于上神的代码

`int byte_cnt = (bits_left - (8 - (bit_idx % 8))) / 8;`，怎么理解？

1. `bit_idx`是什么？当前文件在数据区的偏移量，单位是扇区的数量；它也是在sector-map中占用的bit的数量。
2. `bit_idx % 8`，是不足一个字节的bit的数量。
3. `8 - (bit_idx % 8)`，是补足一个字节需要的bit的数量。
4. `bits_left`是什么？它是一个文件占用的所有扇区的数量。
5. `byte_cnt`是什么？我仍然不清楚。



```c
int i;
	/* clear the first byte */
	for (i = bit_idx % 8; (i < 8) && bits_left; i++,bits_left--) {
		assert((fsbuf[byte_idx % SECTOR_SIZE] >> i & 1) == 1);
		fsbuf[byte_idx % SECTOR_SIZE] &= ~(1 << i);
	}
```

这是在做什么？

### v16

在IPC中，发送方是谁，接收方又是谁？我发现我不能回答这两个问题。

在linux系统中，一切皆文件。按照这个原则，把tty也纳入文件的范畴。读写tty，也应该通过文件系统。

进程读写tty的流程如下：

1. 进程P调用文件系统，告知文件系统，想读写tty。
2. 文件系统通知tty，执行读操作。
3. tty读数据会阻塞，阻塞到什么时候由人什么时候结束在终端的输入决定。文件系统肯定不能因为tty而阻塞，因为还有它还需要处理其他读写操作。
4. P没有收到要读取的数据，不能执行。我的操作系使用IPC实现系统调用。文件系统不阻塞，进程P必须阻塞。
5. 什么时候解除P的阻塞？当tty读取数据结束的时候。

### v17

i版代码运行异常：不能切换终端；键盘输入无响应。

键盘中断正常吗？

使用gdb分别在clock_handle、keyboard_handle设置断点，发现键盘中断、时钟中断都是正常的。

分析TestB中打开文件`int fd_stdin  = open(tty_name, O_RDWR);`的流程：

1. open
2. task_fs
3. case OPEN--->do_open
4. open.c#I_CHAR_SPECIAL
5. task_tty#`case DEV_OPEN`

详细流程：

```c
// 在 open.c 中的 do_open 中
if (imode == I_CHAR_SPECIAL) {
			MESSAGE driver_msg;
			driver_msg.type = DEV_OPEN;
			int dev = pin->i_start_sect;
			driver_msg.DEVICE = MINOR(dev);
			assert(MAJOR(dev) == 4);
			assert(dd_map[MAJOR(dev)].driver_nr != INVALID_DRIVER);
			send_recv(BOTH,
				  dd_map[MAJOR(dev)].driver_nr,
				  &driver_msg);
		}
```

我猜测dd_map[MAJOR(dev)].driver_nr是TASK_TTY。

结合`driver_msg.type = DEV_OPEN;`，我认为，IPC的发送对象是

```c
// 在 tty.c 中的 task_tty 中
// some code
send_recv(RECEIVE, ANY, &msg);
// some code
switch (msg.type) {
		case DEV_OPEN:
			reset_msg(&msg);
			msg.type = SYSCALL_RET;
			send_recv(SEND, src, &msg);
			break;
		case DEV_READ:
			tty_do_read(ptty, &msg);
			break;
		case DEV_WRITE:
			tty_do_write(ptty, &msg);
			break;
		case HARD_INT:
			/**
			 * waked up by clock_handler -- a key was just pressed
			 * @see clock_handler() inform_int()
			 */
			key_pressed = 0;
			continue;
		default:
			dump_msg("TTY::unknown msg", &msg);
			break;
		}
```

更具体一些，执行的代码是：

```c
case DEV_OPEN:
			reset_msg(&msg);
			msg.type = SYSCALL_RET;
			// src 是哪个进程？TASK_FS。
			// 发送给 open.c 中的 open
			send_recv(SEND, src, &msg);
			break;
```

#### 正确的流程

再梳理一次流程：

1. ```c
   char tty_name[] = "/dev_tty1";
   
   	int fd_stdin  = open(tty_name, O_RDWR);
   ```

2. ```c
   // lib/open.c
   PUBLIC int open(const char *pathname, int flags)
   {
   	MESSAGE msg;
   
   	msg.type	= OPEN;
   
   	msg.PATHNAME	= (void*)pathname;
   	msg.FLAGS	= flags;
   	msg.NAME_LEN	= strlen(pathname);
   
   	send_recv(BOTH, TASK_FS, &msg);
   	assert(msg.type == SYSCALL_RET);
   
   	return msg.FD;
   }
   ```

4. ```c
   // /Users/cg/data/code/os/yy-os/osfs09/i/fs/main.c
   PUBLIC void task_fs()
   {
   	printl("Task FS begins.\n");
   
   	init_fs();
   
   	while (1) {
   		send_recv(RECEIVE, ANY, &fs_msg);
   
   		int msgtype = fs_msg.type;
   		int src = fs_msg.source;
   		pcaller = &proc_table[src];
   
   		switch (msgtype) {
   		case OPEN:
        	// do_open，接收一个type是 SYSCALL_RET 的消息，返回一个fd。
   			fs_msg.FD = do_open();
   			break;
   		// some code
   		
       // 这段不起眼的代码，根据msg的type来决定是否解除TestB这类调用文件系统的进程。
       // open解除，read不解除
   		/* reply */
   		if (fs_msg.type != SUSPEND_PROC) {
         // src 是打开中断的进程，例如TestB。
         // do_open 接收到的消息的type是 SYSCALL_RET ，会执行这里的代码。
         // src接收到回复它的消息，解除阻塞。
   			fs_msg.type = SYSCALL_RET;
   			send_recv(SEND, src, &fs_msg);
   		}
   	}
   }
   ```
   
5. ```c
   // /Users/cg/data/code/os/yy-os/osfs09/i/fs/open.c
   PUBLIC int do_open()
   {
   	int fd = -1;		/* return value */
   	// some code
   
   	if (pin) {
   		// some code
   
   		if (imode == I_CHAR_SPECIAL) {
   			MESSAGE driver_msg;
   			driver_msg.type = DEV_OPEN;
   			int dev = pin->i_start_sect;
   			driver_msg.DEVICE = MINOR(dev);
   			assert(MAJOR(dev) == 4);
   			assert(dd_map[MAJOR(dev)].driver_nr != INVALID_DRIVER);
         // 向TTY发送消息，并且接收TTY返回的消息。
         // TTY返回的消息是是什么？
         // 是 msg.type = SYSCALL_RET;
   			send_recv(BOTH,
   				  dd_map[MAJOR(dev)].driver_nr,
   				  &driver_msg);
   		}
   		else if (imode == I_DIRECTORY) {
   			assert(pin->i_num == ROOT_INODE);
   		}
   		else {
   			assert(pin->i_mode == I_REGULAR);
   		}
   	}
   	else {
   		return -1;
   	}
   
   	return fd;
   }
   ```
   
6. ```c
   // /Users/cg/data/code/os/yy-os/osfs09/i/kernel/tty.c
   PUBLIC void task_tty()
   {
   	TTY *	tty;
   	MESSAGE msg;
   
   	// some code
   
   	while (1) {
   		// some code
   
   		send_recv(RECEIVE, ANY, &msg);
   
   		int src = msg.source;
   		assert(src != TASK_TTY);
   
   		TTY* ptty = &tty_table[msg.DEVICE];
   
   		switch (msg.type) {
   		case DEV_OPEN:
   			reset_msg(&msg);
   			msg.type = SYSCALL_RET;
         // src 是 FS
         // 不会阻塞调用文件系统的进程
   			send_recv(SEND, src, &msg);
   			break;
   		case DEV_READ:
         // src 是 FS
         // msg->type = SUSPEND_PROC;
         // 会阻塞调用文件系统的进程
   			tty_do_read(ptty, &msg);
   			break;
   		case DEV_WRITE:
   			tty_do_write(ptty, &msg);
   			break;
   		case HARD_INT:
   			/**
   			 * waked up by clock_handler -- a key was just pressed
   			 * @see clock_handler() inform_int()
   			 */
   			key_pressed = 0;
   			continue;
   		default:
   			dump_msg("TTY::unknown msg", &msg);
   			break;
   		}
   	}
   }
   ```
   
7. 

## 疑问

### 段错误

```c
int main(int argc, char **argv)
{
  	char *str2 = "";
  	*str2 = 'A';
  	
  	return 0;
}
```

神奇。`*str2 = 'A';`会导致段错误。

我的理解是：

1. `""`是一个字符串，作为右值赋值时，它是一个内存地址，等同于`char *tmp = "hello";`中的`tmp`。
2. `*str2`是什么？对一个指针变量使用`*`，意思是获取某个内存地址标识的内存中的数据。简言之，`*str2`是一个数据。对一个数据再赋值，因而出现段错误。
3. 对。就是第2点中理解的那样。(2021-06-16 21:59 补充，耗时)
4. 第3点中的说法，可能不正确。我在《汇编语言程序设计》中找到一句话。



![image-20210616221404247](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/image-20210616221404247.png)

由于这几句话，我不能把`movb $0x41, (%eax)`中的`(%eax)` 理解成内存中的数据。

根据编译器运行结果，我只能猜测，修改目标内存的数据引发了错误。目标内存是：把物理内存地址当ds段的偏移量。

> 在这个问题，我已经耗费了太多时间！大部分时间，我在盯着代码发呆。不知道从哪里着手理解。
>
> 还不如死记硬背这种C语言语句。
>
> 真是得不偿失。投入时间和获得收益完全不成正比！
>
> 到此为止吧。这不是重点问题。



![image-20210616205414949](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/image-20210616205414949.png)

`movb $0x41, (%eax)`，`(%eax)` 是什么？是内存中的数据，简言之，是数据。把立即数复制给另一个数据，当然不行！

`mov %eax, -0x4(%ebp)`中的`-0x4(%ebp)`是内存地址`0x804853c`。

下面两张图是C代码和它对应的汇编代码。

![image-20210616210242484](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/image-20210616210242484.png)



![image-20210616210301150](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/image-20210616210301150.png)



看汇编代码，我看不出笔记中的C代码为什么不正确。

另外一组C代码和对应的汇编代码，能正常运行。和第一幅汇编代码图对应的C代码的差异是，本组使用的是数组，而不是指针。

![image-20210616212442877](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/image-20210616212442877.png)



![image-20210616212410794](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/image-20210616212410794.png)

## 运维

### bochs制作硬盘

```shell
[root@localhost v31]# bximage
========================================================================
                                bximage
  Disk Image Creation / Conversion / Resize and Commit Tool for Bochs
         $Id: bximage.cc 13481 2018-03-30 21:04:04Z vruppert $
========================================================================

1. Create new floppy or hard disk image
2. Convert hard disk image to other format (mode)
3. Resize hard disk image
4. Commit 'undoable' redolog to base image
5. Disk image info

0. Quit

Please choose one [0] 1

Create image

Do you want to create a floppy disk image or a hard disk image?
Please type hd or fd. [hd]

What kind of image should I create?
Please type flat, sparse, growing, vpc or vmware4. [flat]

Choose the size of hard disk sectors.
Please type 512, 1024 or 4096. [512]

Enter the hard disk size in megabytes, between 10 and 8257535
[10] 80

What should be the name of the image?
[c.img]

Creating hard disk image 'c.img' with CHS=162/16/63 (sector size = 512)

The following line should appear in your bochsrc:
  ata0-master: type=disk, path="c.img", mode=flat
[root@localhost v31]#
```



```shell
[root@localhost v31]# ll -lh | grep '.img'
-rw-r--r--. 1 root root 1.5M May 16 16:29 a.img
-rw-r-----. 1 root root  80M May 16 16:32 c.img
```

### fdisk制作分区

书上，fdisk制作分区，选项是cylinder；而现在使用的fdisk，选项是sector。

柱面和扇区的关系。一个柱面有多少个扇区？取决于有多少个磁头。

我使用的是过时的方式制作分区，为了在学习时能从书上得到帮助。

先跟着书本来，等解决主要问题后，再使用最新的分区方式。

注意，过时的方式是`fdisk ./c.img -c=dos -u=cylinders`。

这条命令来自：https://man7.org/linux/man-pages/man8/fdisk.8.html



```shell
[root@localhost v31]# fdisk ./c.img -c=dos -u=cylinders

Welcome to fdisk (util-linux 2.32.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
DOS-compatible mode is deprecated.
Cylinders as display units are deprecated.

Created a new DOS disklabel with disk identifier 0xad02f59b.

Command (m for help): x

Expert command (m for help): c
Number of cylinders (1-1048576, default 10): 162

Expert command (m for help): h
Number of heads (1-255, default 255): 16

Expert command (m for help): r

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (1-4, default 1):
First cylinder (1-162, default 1):
Last cylinder, +cylinders or +size{K,M,G,T,P} (1-162, default 162): 20

Created a new partition 1 of type 'Linux' and of size 9.8 MiB.

Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): e
Partition number (2-4, default 2):
First cylinder (21-162, default 21):
Last cylinder, +cylinders or +size{K,M,G,T,P} (21-162, default 162):

Created a new partition 2 of type 'Extended' and of size 69.9 MiB.

Command (m for help): n
All space for primary partitions is in use.
Adding logical partition 5
First cylinder (21-162, default 21):
Last cylinder, +cylinders or +size{K,M,G,T,P} (21-162, default 162): 60

Created a new partition 5 of type 'Linux' and of size 19.7 MiB.

Command (m for help): n
All space for primary partitions is in use.
Adding logical partition 6
First cylinder (61-162, default 61):
Last cylinder, +cylinders or +size{K,M,G,T,P} (61-162, default 162): 90

Created a new partition 6 of type 'Linux' and of size 14.8 MiB.

Command (m for help): n
All space for primary partitions is in use.
Adding logical partition 7
First cylinder (91-162, default 91):
Last cylinder, +cylinders or +size{K,M,G,T,P} (91-162, default 162): 132

Created a new partition 7 of type 'Linux' and of size 20.7 MiB.

Command (m for help): n
All space for primary partitions is in use.
Adding logical partition 8
First cylinder (133-162, default 133):
Last cylinder, +cylinders or +size{K,M,G,T,P} (133-162, default 162): 160

Created a new partition 8 of type 'Linux' and of size 13.8 MiB.

Command (m for help): n
All space for primary partitions is in use.
Adding logical partition 9
First cylinder (161-162, default 161):
Last cylinder, +cylinders or +size{K,M,G,T,P} (161-162, default 162):

Created a new partition 9 of type 'Linux' and of size 976.5 KiB.

Command (m for help): p
Disk ./c.img: 79.8 MiB, 83607552 bytes, 163296 sectors
Geometry: 16 heads, 63 sectors/track, 162 cylinders
Units: cylinders of 1008 * 512 = 516096 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xad02f59b

Device     Boot Start   End Cylinders   Size Id Type
./c.img1            1    20        20   9.8M 83 Linux
./c.img2           21   162       143  69.9M  5 Extended
./c.img5           21    60        40  19.7M 83 Linux
./c.img6           61    90        30  14.8M 83 Linux
./c.img7           91   132        42  20.7M 83 Linux
./c.img8          133   160        28  13.8M 83 Linux
./c.img9          161   162         2 976.5K 83 Linux

Command (m for help): t
Partition number (1,2,5-9, default 9): 5
Hex code (type L to list all codes): L

 0  Empty           24  NEC DOS         81  Minix / old Lin bf  Solaris
 1  FAT12           27  Hidden NTFS Win 82  Linux swap / So c1  DRDOS/sec (FAT-
 2  XENIX root      39  Plan 9          83  Linux           c4  DRDOS/sec (FAT-
 3  XENIX usr       3c  PartitionMagic  84  OS/2 hidden or  c6  DRDOS/sec (FAT-
 4  FAT16 <32M      40  Venix 80286     85  Linux extended  c7  Syrinx
 5  Extended        41  PPC PReP Boot   86  NTFS volume set da  Non-FS data
 6  FAT16           42  SFS             87  NTFS volume set db  CP/M / CTOS / .
 7  HPFS/NTFS/exFAT 4d  QNX4.x          88  Linux plaintext de  Dell Utility
 8  AIX             4e  QNX4.x 2nd part 8e  Linux LVM       df  BootIt
 9  AIX bootable    4f  QNX4.x 3rd part 93  Amoeba          e1  DOS access
 a  OS/2 Boot Manag 50  OnTrack DM      94  Amoeba BBT      e3  DOS R/O
 b  W95 FAT32       51  OnTrack DM6 Aux 9f  BSD/OS          e4  SpeedStor
 c  W95 FAT32 (LBA) 52  CP/M            a0  IBM Thinkpad hi ea  Rufus alignment
 e  W95 FAT16 (LBA) 53  OnTrack DM6 Aux a5  FreeBSD         eb  BeOS fs
 f  W95 Ext'd (LBA) 54  OnTrackDM6      a6  OpenBSD         ee  GPT
10  OPUS            55  EZ-Drive        a7  NeXTSTEP        ef  EFI (FAT-12/16/
11  Hidden FAT12    56  Golden Bow      a8  Darwin UFS      f0  Linux/PA-RISC b
12  Compaq diagnost 5c  Priam Edisk     a9  NetBSD          f1  SpeedStor
14  Hidden FAT16 <3 61  SpeedStor       ab  Darwin boot     f4  SpeedStor
16  Hidden FAT16    63  GNU HURD or Sys af  HFS / HFS+      f2  DOS secondary
17  Hidden HPFS/NTF 64  Novell Netware  b7  BSDI fs         fb  VMware VMFS
18  AST SmartSleep  65  Novell Netware  b8  BSDI swap       fc  VMware VMKCORE
1b  Hidden W95 FAT3 70  DiskSecure Mult bb  Boot Wizard hid fd  Linux raid auto
1c  Hidden W95 FAT3 75  PC/IX           bc  Acronis FAT32 L fe  LANstep
1e  Hidden W95 FAT1 80  Old Minix       be  Solaris boot    ff  BBT
Hex code (type L to list all codes): L

 0  Empty           24  NEC DOS         81  Minix / old Lin bf  Solaris
 1  FAT12           27  Hidden NTFS Win 82  Linux swap / So c1  DRDOS/sec (FAT-
 2  XENIX root      39  Plan 9          83  Linux           c4  DRDOS/sec (FAT-
 3  XENIX usr       3c  PartitionMagic  84  OS/2 hidden or  c6  DRDOS/sec (FAT-
 4  FAT16 <32M      40  Venix 80286     85  Linux extended  c7  Syrinx
 5  Extended        41  PPC PReP Boot   86  NTFS volume set da  Non-FS data
 6  FAT16           42  SFS             87  NTFS volume set db  CP/M / CTOS / .
 7  HPFS/NTFS/exFAT 4d  QNX4.x          88  Linux plaintext de  Dell Utility
 8  AIX             4e  QNX4.x 2nd part 8e  Linux LVM       df  BootIt
 9  AIX bootable    4f  QNX4.x 3rd part 93  Amoeba          e1  DOS access
 a  OS/2 Boot Manag 50  OnTrack DM      94  Amoeba BBT      e3  DOS R/O
 b  W95 FAT32       51  OnTrack DM6 Aux 9f  BSD/OS          e4  SpeedStor
 c  W95 FAT32 (LBA) 52  CP/M            a0  IBM Thinkpad hi ea  Rufus alignment
 e  W95 FAT16 (LBA) 53  OnTrack DM6 Aux a5  FreeBSD         eb  BeOS fs
 f  W95 Ext'd (LBA) 54  OnTrackDM6      a6  OpenBSD         ee  GPT
10  OPUS            55  EZ-Drive        a7  NeXTSTEP        ef  EFI (FAT-12/16/
11  Hidden FAT12    56  Golden Bow      a8  Darwin UFS      f0  Linux/PA-RISC b
12  Compaq diagnost 5c  Priam Edisk     a9  NetBSD          f1  SpeedStor
14  Hidden FAT16 <3 61  SpeedStor       ab  Darwin boot     f4  SpeedStor
16  Hidden FAT16    63  GNU HURD or Sys af  HFS / HFS+      f2  DOS secondary
17  Hidden HPFS/NTF 64  Novell Netware  b7  BSDI fs         fb  VMware VMFS
18  AST SmartSleep  65  Novell Netware  b8  BSDI swap       fc  VMware VMKCORE
1b  Hidden W95 FAT3 70  DiskSecure Mult bb  Boot Wizard hid fd  Linux raid auto
1c  Hidden W95 FAT3 75  PC/IX           bc  Acronis FAT32 L fe  LANstep
1e  Hidden W95 FAT1 80  Old Minix       be  Solaris boot    ff  BBT
Hex code (type L to list all codes): 99

Changed type of partition 'Linux' to 'unknown'.

Command (m for help): a
Partition number (1,2,5-9, default 9): 5

The bootable flag on partition 5 is enabled now.

Command (m for help): p
Disk ./c.img: 79.8 MiB, 83607552 bytes, 163296 sectors
Geometry: 16 heads, 63 sectors/track, 162 cylinders
Units: cylinders of 1008 * 512 = 516096 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xad02f59b

Device     Boot Start   End Cylinders   Size Id Type
./c.img1            1    20        20   9.8M 83 Linux
./c.img2           21   162       143  69.9M  5 Extended
./c.img5   *       21    60        40  19.7M 99 unknown
./c.img6           61    90        30  14.8M 83 Linux
./c.img7           91   132        42  20.7M 83 Linux
./c.img8          133   160        28  13.8M 83 Linux
./c.img9          161   162         2 976.5K 83 Linux

Command (m for help): w
The partition table has been altered.
Syncing disks.

[root@localhost v31]#
```

### 计算机启动方式

目前的常用的启动方式有 `BIOS & MBR`, `UEFI & GPT`. 其中 `MBR` 和 `GPT` 是硬盘分区的两种格式.通常是 `BIOS` 配合 `MBR`,`UEFI` 配合 `GPT`.

在早期的 `IBM PC` 兼容机中，`BIOS` 配合 `MBR` 磁盘分区进行启动.

##### BIOS 和 MBR 是什么?

`BIOS` 是 Basic Input/Output System 的缩写,是一种固件程序,一般不能修改. `MBR` 是 Master Boot Record 的缩写, 这是磁盘上一个特定扇区的名称,通常是HDD(0,0,1),即第一个磁头,第一个磁道,第一个扇区,也使用 `MBR` 来表示这种磁盘分区方式,以及使用`MBR`来代称第一个扇区(一个扇区512个字节)的前446个字节,这446个字节是引导操作系统的代码.

##### `MBR` 扇区的结构

如下:

- 0x000-0x1B7 : 共 440 个字节, 引导程序
- 0x1B8-0x1BB : 共 4 个字节, 记录选用磁盘标志
- 0x1BC-0x1BD : 一般为空值, 0x0000
- 0x1BE-0x1FD : 记录标准 MBR 格式的分区表规划
- 0x1FE-0x1FF : 0x55AA MBR 有效标志位

### PBT实例分析

```shell
																							00 01  ........."......
000001c0: 01 00 83 0F 3F 13 3F 00 00 00 81 4E 00 00
```

英文版在这里

https://en.wikipedia.org/wiki/Master_boot_record

不容易看懂。

中文版在这里

https://zh.wikipedia.org/wiki/%E4%B8%BB%E5%BC%95%E5%AF%BC%E8%AE%B0%E5%BD%95



## 工具

### vim+ctags+Taglist

```shell
函数跳转
ctrl + ]
跳回调用函数的地方
ctrl + t
在vim中打开Taglist
:Tlist
```

Markdown 数学

https://www.jianshu.com/p/4460692eece4

### linux命令

```shell
# 在当前目录下查找包含hd_info的字符串。
# 不要使用-v，否则，会打印出大量无关查询结果
grep -r 'hd_info' ./
# 查看主设备号
cat /proc/devices
# 查看当前设备的主次设备号
ls -l /dev
# 在主机之间复制文件
scp ./tool.sh root@172.16.64.132:/home/cg/yuyuan-os/osfs09/h/
# 查找出进程并且kill
ps -ef | grep bochs | grep -v grep | awk '{print $2}' | xargs kill -9

```



#### awk、xargs、grep -v

这几个命令组合起来，用途大，容易写错。注意下面的错误写法和正确写法。

```shell
# grep -v 的用法，使用两个grep并且用管道连接起来。
[root@localhost j]# ps -ef | grep bochs | grep -v grep | awk {print $2}
awk: cmd. line:1: {print
awk: cmd. line:1:       ^ unexpected newline or end of string
[root@localhost j]# ps -ef | grep bochs | grep -v grep | awk '{print $2}'
awk: cmd. line:1: {print $2}�
awk: cmd. line:1:           ^ invalid char '�' in expression
[root@localhost j]# ps -ef | grep bochs | grep -v grep | awk '{print $2}'
296499
# awk "{print $2}" 是错误的，并没有只输出第二列数据，而是输出了全部数据。
[root@localhost j]# ps -ef | grep bochs | grep -v grep | awk "{print $2}"
cg        296499  296458  0 18:08 pts/0    00:00:00 bochs -f bochsrc
[root@localhost j]# ps -ef | grep bochs | grep -v grep | awk '{print $2}' | xargs kill -9
[root@localhost j]# ps -ef | grep bochs | grep -v grep | awk '{print $2}'
```



```c
// grep 显示行号，-n
// grep 递归查找目录，也就是连子目录也查找，-r
[root@localhost i]# grep -rn "RESUME" ./*
Binary file ./a.img matches
./fs/main.c:63:		case RESUME_PROC:
./fs/main.c:109:		case RESUME_PROC:
./include/sys/const.h:143:	SUSPEND_PROC, RESUME_PROC,
./kernel/tty.c:280:
```



### gdb

1. 1字节（byte）=8位（bit）
2. 在16位系统中，1字（word）=2字节（byte）=16位（bit）
3. 在32位系统中，1字（word）=4字节（byte）=32位（bit）
4. 在64位系统中，1字（word）=8字节（byte）=64位（bit）

在64位电脑上使用gdb，默认是用64位下的进制转换，也就是，1字 = 8字节。

```shell
(gdb) help x/FMT
Examine memory: x/FMT ADDRESS.
ADDRESS is an expression for the memory address to examine.
FMT is a repeat count followed by a format letter and a size letter.
Format letters are o(octal), x(hex), d(decimal), u(unsigned decimal),
  t(binary), f(float), a(address), i(instruction), c(char), s(string)
  and z(hex, zero padded on the left).
Size letters are b(byte), h(halfword), w(word), g(giant, 8 bytes).
The specified number of objects of the specified size are printed
according to the format.  If a negative number is specified, memory is
examined backward from the address.

Defaults for format and size letters are those previously used.
Default count is 1.  Default address is following last thing printed
with this command or "print".
(gdb) x /1wx fsbuf+255
0x6000ff:	0x000000ff
(gdb) x /1hx fsbuf+255
0x6000ff:	0x00ff
(gdb) x /1hx fsbuf+256
0x600100:	0x0000
(gdb) s
249			fsbuf[i] |= (1 << j);
(gdb) p i
$4 = 256
(gdb) s
248		for (j = 0; j < nr_sects % 8; j++)
(gdb) s
257		WR_SECT(ROOT_DEV, 2 + sb.nr_imap_sects);
(gdb) x /1hx fsbuf+256
0x600100:	0x0001
(gdb) x /1hx fsbuf+257
0x600101:	0x0000
(gdb) x /1hx fsbuf+258
0x600102:	0x0000
(gdb) x /1bx fsbuf+258
0x600102:	0x00
```

在上面的gdb断点数据中，`x /1hx`打印的是半个字的数据，长度却是16个字节`0x0001`。这个结果，显然不是32位计算机下“1字=2个字节；半个字=1个字节”的进制转换的结果。

使用`help x`能查看gdb的x的使用方法。

### git

```shell
git rm -r --cached .
git add .
git commit -m 'update .gitignore'
```



## 博客

http://www.techgogogo.com/page/2/

在你阅读Linux内核源码之前，先要确保已经掌握了csapp（Computer System A Programmer's Perspective）和ostep ( Operating system three easy pieces )的内容。

然后先拿mit的xv6。

接下来你需要安装一个开发环境以便你调试源码，推荐用docker建立一个本地镜像，你在本地的所有操作都会同步到容器中，这样你可以一边在本地用ide操作源码，一边在docker环境中进行编译、执行。

能使用docker吗？不是需要安装bochs吗？在docker中不能安装啊。

先读ostep、csapp，然后推荐读一下bach的 “the design of unix operating system”

真想看代码也配一本好书，比如professional linux kernel architecture. 我看了一点太厚了扔一边了...不过写的确实很好

MIT6.006（好像是这个数字）的算法课程，跟着那个印度裔老师学了下这本书，但是终究没坚持下来。后来我发现了coursera，从学校图书馆借来了算法第四版，总算是把基础算法过了一遍，但是算法导论后来就没怎么翻了。

后来研究生上课也有算法分析，我又看了下MIT6.046，还比较装逼地看了下MIT6.851，记得第一个高级数据结构是讲的version control的数据结构。

https://bellard.org/

一个外国神人的博客，上面有很多神奇的项目：在浏览器运行操作系统的虚拟机、把java代码转换成C代码的软件等。

## 书籍

《Linux内核设计》

## 资料

汇编指令详细解释

http://css.csail.mit.edu/6.858/2014/readings/i386/REP.htm	

电子书在线阅读网站

https://www.bookstack.cn/explore?cid=28&tab=popular

Linux C编程一站式学习 宋劲杉

https://akaedu.github.io/book/

## 总结

### 2021-05-16 18:18

```c
PRIVATE void print_identify_info(u16* hdinfo)
{
        int i, k;
        char s[64];

        struct iden_info_ascii {
                int idx;
                int len;
                char * desc;
        } iinfo[] = {{10, 20, "HD SN"}, /* Serial number in ASCII */
                     {27, 40, "HD Model"} /* Model number in ASCII */ };

        for (k = 0; k < sizeof(iinfo)/sizeof(iinfo[0]); k++) {
                char * p = (char*)&hdinfo[iinfo[k].idx];
                for (i = 0; i < iinfo[k].len/2; i++) {
                        s[i*2+1] = *p++;
                        s[i*2] = *p++;
                }
                s[i*2] = 0;
                printl("%s: %s\n", iinfo[k].desc, s);
        }

        int capabilities = hdinfo[49];
        printl("LBA supported: %s\n",
               (capabilities & 0x0200) ? "Yes" : "No");

        int cmd_set_supported = hdinfo[83];
        printl("LBA48 supported: %s\n",
               (cmd_set_supported & 0x0400) ? "Yes" : "No");

        int sectors = ((int)hdinfo[61] << 16) + hdinfo[60];
        printl("HD size: %dMB\n", sectors * 512 / 1000000);
}
```

理解不了 `int sectors = ((int)hdinfo[61] << 16) + hdinfo[60];`，还有下面的

```c
char * p = (char*)&hdinfo[iinfo[k].idx];
                for (i = 0; i < iinfo[k].len/2; i++) {
                        s[i*2+1] = *p++;
                        s[i*2] = *p++;
                }
                s[i*2] = 0;
```

消耗了1个小时55分，仍没有解决问题。

为什么花了这么长时间？

1. 不理解。盯着代码看，却没有任何猜想。
2. 找相关的书，浏览。
3. 查网络，关键词不正确，反倒看了很多不相关的知乎帖子。这，浪费了最多时间。

最后，再用关键词搜索，找到了文档。

### 2021-05-16 23:04

```c
char * p = (char*)&hdinfo[iinfo[k].idx];
                for (i = 0; i < iinfo[k].len/2; i++) {
                        s[i*2+1] = *p++;
                        s[i*2] = *p++;
                }
                s[i*2] = 0;
```

理解不了这段代码。耗费了1小时14分。

无法用大小端法理解。

小端法：数值的低位放到内存的低地址。

大端法：数值的低位放到内存的高地址。

无法理解，是因为，

1. `int sectors = ((int)hdinfo[61] << 16) + hdinfo[60];`，数值的高位在高地址。这说明，硬盘数据，是小端法。
2. 而上面这段，把硬盘中的数据的顺序颠倒了存储到内存中。不知道为什么要这样做。这不符合小端法。
3. 作者的代码，无法运行。也是因为，内存太大。编译时去掉了`-g`就可以运行了。
4. 对照上面的两张图，可以看出为什么要颠倒了。不颠倒，字符顺序不对。
5. 从硬盘中读取的数据，单位是字，顺序应该颠倒一下。但在内存中处理，仍可以用字节做单位。
   1. 这就是`iinfo[k].len/2`的来历。
   2. 也是`*p++`能使用的原因。

### 2021-05-17 08:59

使用fdisk制作分区。耗费了2个小时3分。

为什么需要这么多时间？

1. 看书上的分区命令。好像理解不了。
   1. fdisk只是一个工具，不需要理解，也能使用。应该先用。
2. 《一个操作系统的实现》和《操作系统真相还原》结合看，后者对分区原理写得太详细，不怎么好理解。
   1. 我的目的是，制作分区，然后继续实现文件系统。重点不是理解分区。
   2. 而且，我看分区理论知识，是因为我觉得我不知道怎么分区。这是缘木求鱼。
3. 分区命令不能用。
   1. 不能用？对于黑盒子工具，不能用，最佳方式，是去搜索，而不是自己瞎尝试。

### 2021-05-17 10:21

新硬盘的`0x1be`和`0x1b7`。书上说，新硬盘的前`0x1be`个字节都是0，可我用xxd查看到的却不是。

纠结，我以为创建硬盘出了问题。用”硬盘 0x1b7"搜索后，发现，书上的不正确。

### 2021-05-17 11:57

理解下面的表格中的“文件系统标志位”。

| 偏移 | 长度（字节） | 意义                                                         |
| :--: | :----------: | :----------------------------------------------------------- |
| 00H  |      1       | 分区状态：00-->非活动分区；80-->活动分区； 其它数值没有意义  |
| 01H  |      1       | 分区起始磁头号（HEAD），用到全部8位                          |
| 02H  |      2       | 分区起始扇区号（SECTOR），占据02H的位0－5； 该分区的起始磁柱号（CYLINDER），占据 02H的位6－7和03H的全部8位 |
| 04H  |      1       | [文件系统](https://zh.wikipedia.org/wiki/文件系统)标志位     |
| 05H  |      1       | 分区结束磁头号（HEAD），用到全部8位                          |
| 06H  |      2       | 分区结束扇区号（SECTOR），占据06H的位0－5； 该分区的结束磁柱号（CYLINDER），占据 06H的位6－7和07H的全部8位 |
| 08H  |      4       | 分区起始相对扇区号                                           |
| 0CH  |      4       | 分区总的扇区数                                               |

花了很长时间。我一直以为偏移`00H`位置才是文件系统标志位。

为什么会如此？

1. 实例分析中，`80`和文件系统标志位非常相似。
2. 上面的表格的解释文字中说，偏移`04H`位置是`FAT12`这类文件。岂不知，我要写的操作系统，使用的自创的文件系统，标志类型是`99`。
3. 偏移量
   1. 初始值，是0。记住了，是0，不是1。
   2. 偏移量是N，意思是，从0开始计数，一直数到第N个。第N个元素，就是偏移量为N的目标元素。
   3. 再补充。偏移量是N，不是说，在目标元素的前面有N个元素，而是说，目标元素是第N个元素，起点是0。
   4. 也就是说，偏移量为N的元素，是第N+1个元素。

### 2021-05-17 14:06

MBR；硬盘分区。有点难理解。

看郑上神的书，和维基百科中的关键语句，才理解。于上神的书，高估了读者，更适合作为“纲”。

这个关键语句(？哪个关键语句？笔记残缺不全）告诉我，每个EBR，包含两部分：

1. 一个主分区或逻辑分区。
2. 下一个扩展分区的元数据。
3. 其实，都是元数据。

耗费了1个小时22分。

为什么花了这么多时间？

1. 使用grep。实际消耗时间应该很少。
2. 抛开书本看代码。看不懂。所花时间，应该也不多。
3. 乱查各种网络资料。不应该，两本书和维基百科，应该是主要阅读材料。
4. 没有仔细看书？
5. 分心，杂念比较多。

### 2021-05-17 14:45

主次设备号。大概看明白。

耗费大概半个小时。

为啥慢？

1. 扫了眼于上神的书，没看明白，乱找其他资料。这知识点不流行吧，资料少，不敢信。
2. 仔细看于上神的书，看明白了。

### 2021-05-17 16:55

看硬盘分区表代码，没看懂。不是全部看不懂，有几个关键的地方看不懂。

耗费1个小时19分。

能在5月20号完成文件系统就好了。难度不在代码量和思考量多，而在不理解，另外就怕遇到IPC那样耗费很多时间的问题。

仿写boss直聘，需要不停地思考各种不难但也需要想的问题，代码量非常大，我的手指敲击键盘都觉得有点疼。

### 2021-05-17 17:56

重读《操作系统真相还原》书中的硬盘分区表章节。看懂了大部分，但只记住了很少。

总扩展分区能划分成多个子扩展分区，每个子扩展分区都相当于一个硬盘，都有EBR、逻辑分区。每个子扩展分区之间通过EBR中的DPT联系起来。

耗费了1个小时。对理解于上神的代码，有用吗？

### 2021-05-17 19:46

理解“每个扩展分区最多有16个逻辑分区”。

耗费了34分钟。

这句话和其他资料矛盾，所以，我对这句话进行了自我理解。

### 2021-05-17 20:13

修复内核太大不能运行的问题；又看了很久那些宏，仍然看不懂。没有啥办法，只能多看几次。

耗费时间57分。

### 2021-05-17 23:06

成果是上面的v7。消耗了31分钟。

看吧，进展虽慢，但花了时间，还是往前推进了啊。所以，不要浪费时间。

### 2021-05-17 23:39

成果是上面的v9。消耗了33分钟。

那个宏，根据分区号计算驱动器号。不对，没有分区号这个叫法，应该叫“次设备号”。

为什么能够理解？

1. 断点看到过计算结果是0。
2. 先理解了于上神设计的主次设备号方案，代入具体数值计算。
3. 联想到device寄存器的第4位。

盯着代码看，效果很差的。理解不了的时候，要多假设，去思考，盯着代码、不主动有条理地想、等待顿悟，效果很差。

### 2021-05-25 09:25

看八天前写的笔记。花了比较多时间才看懂v3笔记。笔记写得不完整导致如此。

浪费了整整八天。

亡羊补牢。

### 2021-05-25 10:27

理解了v9中的宏并且补充。消耗时间38分钟。

### 2021-05-25 12:13

费时1个小时。看“用代码遍历所有分区”代码，看懂了一点，收获很小。

`open_cnt`记录硬盘打开次数。

我在做什么？

1. 看书中代码。书中代码，看起来不舒服，也许是因为看到一个结构体需要翻到其他地方。
2. 看不明白。盯着看。
   1. 看不明白，就应该主动思考。这块的代码在做什么？

### 2021-05-25 16:27

v10。理解不了。耗时1小时12分。

hdbuf是一个u8数组，转化为u16指针，如何理解？

`int *p`，p是一个内存地址，任何类型的指针的值都是一个内存地址。`int *`规定了，内存地址指向的那个内存开始，4个字节即连续32个bit都是`int *p`中存储的数据。

### 2021-05-25 18:08

理解v10中的指针。耗时42分。

事实：

1. Indentify 查询到的硬盘参数，单位是字。什么意思？如果查询到的结果存储在`u16 hdinfo`中，hdinfo[0]的值是一个字的数据。
2. `int capabilities = hdinfo[49];`，hdinfo[49]的值是一个字的数据。

```c
PRIVATE u8              hdbuf[SECTOR_SIZE * 2];

PRIVATE void hd_identify(int drive)
{
        struct hd_cmd cmd;
        cmd.device  = MAKE_DEVICE_REG(0, drive, 0);
        cmd.command = ATA_IDENTIFY;
        hd_cmd_out(&cmd);
        interrupt_wait();
        port_read(REG_DATA, hdbuf, SECTOR_SIZE);

        print_identify_info((u16*)hdbuf);

        u16* hdinfo = (u16*)hdbuf;

        hd_info[drive].primary[0].base = 0;
        /* Total Nr of User Addressable Sectors */
        // 小端法
        // 为什么要使用hdbuf而不是直接使用hddinfo？
        // 为什么要把hdinfo声明成为u16 *类型指针？
        // hdinfo[60]和hdinfo[61]，都是一个字节的数据而不是一个字。
        hd_info[drive].primary[0].size = ((int)hdinfo[61] << 16) + hdinfo[60];
}
```

`hdbuf`本来是一个u8类型数组，后来，将它强制转换成u16类型数组。

### 2021-05-25 21:17

耗时47分。

紧接上面的那个总结，回答几个问题。

1. 为什么要使用hdbuf而不是直接使用hddinfo？
   1. 因为hdinfo是一个char类型数组，而在最后一句，需要使用一个u16类型数组。
   2. 在最后一句使用(u16 *)hdbuf代替hdinfo行不行？我不知道，可以修改代码然后运行一下。

1. 为什么要把hdinfo声明成为u16 *类型指针？
   1. 因为，硬盘参数，读写数据的单位是一个字二个字节。读取到的硬盘参数（原始数据），单位是字节。
   2. 需要重组数据，把它们重组成按“字”排列。

用具体的例子来理解。

```c
int main(int argc, char **argv)
{
  	char buf[4] = {0x00, 0x01, 0x02, 0x03};
  	short *buf2 = (short *)buf;
  
  	return 0;
}
```

用gdb断点查看到的数据如下：

```shell
(gdb) s
37		short *buf2 = (short *)buf;
(gdb) p (unsigned char)buf[3]
$8 = 3 '\003'
(gdb) s
40		return 0;
(gdb) p buf2[0]
$9 = 256
(gdb) p buf2[1]
$10 = 770
(gdb) p /t buf2[0]
$11 = 100000000
(gdb) p /t buf[0]
$12 = 0
(gdb) p /t buf[1]
$13 = 1
(gdb) p /t buf2[1]
$14 = 1100000010
(gdb) p /t buf[2]
$15 = 10
(gdb) p /t buf[3]
$16 = 11
```

对`short *buf2 = (short *)buf;`的理解：

1. buf是一个内存地址A，(short *)buf的意思是，从内存A开始，每2个字节的内存存储一个buf2的数据。
2. 注意了，从内存A开始的那一片内存中，数据没有变化，仍然可以用“字节”为单位读写。任何时候，内存中的数据都能够用“字节”为单位读写。
3. 所以，即使执行了`(short *)buf`，buf[0]的值也和执行`(short *)buf`前相同。
4. 但是，buf2[0]和buf[0]不同。buf2[0]是buf[0]、buf[1]的组合在一起的结果。
5. 使用`p /t buf2[0]`打印出buf2[0]，结果是：`100000000`。内存地址是右低左高，在数组中，内存地址是左低右高（和数组索引一致）。

### 2021-05-25 21:26

理解`#疑问>段错误`。耗时7分钟。

### 2021-05-25 21:56

```c
for (k = 0; k < sizeof(iinfo)/sizeof(iinfo[0]); k++) {
  // 明白了这句。hdinfo本来是u16类型数组，经过这句后，变成了char类型数组。
  // *p 是字符。
  char * p = (char*)&hdinfo[iinfo[k].idx];
  for (i = 0; i < iinfo[k].len/2; i++) {
    s[i*2+1] = *p++;
    s[i*2] = *p++;
  }
  s[i*2] = 0;
  printl("%s: %s\n", iinfo[k].desc, s);
}
```

收获见上面的代码中的注释。

耗时30分钟。

### 2021-05-26 09:13

理解v11的代码，耗时58分。

效果很差。

我在做什么？盯着代价看，慢慢理解。除此之外，还能怎么办？遇到理解不了的问题，很容易分心。

我打算写出每个理解不了的问题，逐个理解。

### 2021-05-26 10:25

理解v11的代码，耗时32分。

又看懂了一点，关键是理解了`get_part_table`。

不应该在这里有疑问。

### 2021-05-26 12:25

理解v11的代码，耗时1小时19分。真正的有效时间大概是30分钟。其他时间，盯着代码看，在代码中跳来跳去，等待灵感或顿悟。还有一部分时间，我在理解其他代码。为什么要获取分区，为什么要在获取主分区时获取逻辑分区。

收获是，明白了`partition(device + dev_nr, P_EXTENDED);`不能修改成`partition(dev_nr, P_EXTENDED);`。

怎么理解的？

1. 明确问题。
2. 代入实例，用笔算。
   1. `partition(device + dev_nr, P_EXTENDED);`能不能修改成`partition(dev_nr, P_EXTENDED);`？
   2. 很简单。把两种情况都计算一次，就知道结果了。
   3. 步骤有点多，用心算纯属浪费时间。
3. 不盯着代码看，不心算。

### 2021-05-27 08:57

理解v11的代码的`else if (style == P_EXTENDED)`，耗时1个小时42分。

效果很差，收获为零。听着歌，再加上一些事情，分心。

### 2021-05-28 09:18

理解v11的代码，耗时1小时4分钟（没有统计之前的时间）。

为什么这么慢？因为前天知道的坏消息，再加上我比较容易分心，然后，这个知识点，我一直没有看清楚。

几个术语：

1. 总扩展分区。硬盘最多能划分为四个主分区。每个主分区都可以当作扩展分区，但只能有一个。这个扩展分区就是总扩展分区。
2. 子扩展分区。
   1. 总扩展分区中有一个EBR，EBR中的DPT有两个磁盘分区表项。第一个表项指向逻辑分区；第二个表项指向扩展分区，这个扩展分区叫做子扩展分区。
   2. 子扩展分区又有一个EBR。EBR的结构和第1点中的EBR的结构相同。
3. 逻辑分区。

总扩展分区的绝对LBA地址 = 0 + 总扩展分区的偏移量。

子扩展分区的绝对LBA地址 = 总扩展分区的绝对LBA地址 + 子扩展分区的偏移量。

逻辑分区的绝对LBAR地址 = 子扩展分区的绝对LBA地址 + 逻辑分区的偏移量。

### 2021-05-28 17:39

重新理解遍历硬盘分区的代码的每个细节，理解速度很慢，思维非常不连贯，各种杂念。

耗时58分。

### 2021-05-28 19:45

运行`/home/cg/yuyuan-os/osfs09/c`中的代码时，出现错误：`bios_printf: unknown format`。

原因是：1. Kernel.bin过大；2. Kernel.bin和loader.bin的内存分布错误。

耗时1个小时20分。一部分时间消耗在看代码，主要时间消耗在解决上面的问题。

怎么解决问题的？对比正确的代码，再加上经验。

唉，我已经开始遗忘之前的知识了：

1. 实模式下的寻址方式。
2. objdump的使用。
3. 加载内核bx溢出。

经验：

1. 在写操作系统中，不是知识点，只是报错信息，别指望搜索引擎。写操作系统是个小众爱好，不像其他语言，用报错信息能直接搜到解决方案。
2. bochs断点中的机器码，怎么在反汇编文件中寻找？
   1. 进入保护模式，更准确地说，进入内核后，直接根据`(0) [0x000000030405] 0008:0000000000030405 `中的`30405`寻找。
   2. 在实模式中，不知道怎么在反汇编文件中寻找。

### 2021-05-28 22:28

文件系统的实现。耗时1小时43分。

收效甚微：

1. 乱找资料。于上神的文件系统设计得非常简单，不理解他的，想从其他书理解他的系统，是徒劳。不要乱找，多看于上神的书和代码。
2. 文件系统的数据结构太多成员，不知道它们的用途，也记不住。
   1. 其实，这不是问题。当初，进程表的数据结构也很多。我怎么掌握的？多看，回忆，对照书上的资料，能写出所有成员。
   2. 可以看所有代码去理解每个成员的作用。

烦恼。耗时这么久，只学了操作系统内核的一点点皮毛。边学还在边忘记。

### 2021-05-29 08:28

文件系统的实现。耗时1小时23分。

我做了什么？

1. 用vim看代码。
2. 看“编译器相关”招聘岗位。为啥看？我不知道学习编译器有没有用。
3. 制作《C编译器剖析》书签。

看不懂文件系统的实现，所以我就转而做其他事情。

看不懂怎么办？只能继续看。可以放一放，但不能一直放着不看。我最终一定会看懂。从学操作系统以来，遇到那么多看不懂的代码，不也一点点看懂了吗？

比较可惜，“那么多看不懂的代码”，我却不记得它们具体是什么代码。

### 2021-05-29 09:56

看懂了下面的代码。耗时1个小时22分。

```c
// /Users/cg/data/code/os/yy-os/osfs09/e/fs/open.c
/*****************************************************************************
 *                                alloc_imap_bit
 *****************************************************************************/
/**
 * Allocate a bit in inode-map.
 * 
 * @param dev  In which device the inode-map is located.
 * 
 * @return  I-node nr.
 *****************************************************************************/
PRIVATE int alloc_imap_bit(int dev)
{
	int inode_nr = 0;
	int i, j, k;

	// 启动扇区和超级块各自占用一个扇区
  int imap_blk0_nr = 1 + 1; /* 1 boot sector & 1 super block */
	struct super_block * sb = get_super_block(dev);
	
  // 遍历扇区--->遍历扇区的每个字节--->检查每个字节的每个bit是不是都是1--->如果不是，找到这个字节中不是1的那个bit---->把那个bit设置成1
	for (i = 0; i < sb->nr_imap_sects; i++) {
    // 读取一个扇区到fsbuf中
		RD_SECT(dev, imap_blk0_nr + i);
		// 遍历扇区
		for (j = 0; j < SECTOR_SIZE; j++) {
      // 以字节为单位进行检查
			/* skip `11111111' bytes */
			if (fsbuf[j] == 0xFF)
				continue;
			// 以bit为单位检查一个字节
			/* skip `1' bits */
			for (k = 0; ((fsbuf[j] >> k) & 1) != 0; k++) {}
			
      // 目标bit在硬盘中的位置。
			/* i: sector index; j: byte index; k: bit index */
			inode_nr = (i * SECTOR_SIZE + j) * 8 + k;
      // 把最新inode-map写入硬盘
			fsbuf[j] |= (1 << k);
			// 把fsbuf写入硬盘的细节没看明白
			/* write the bit to imap */
			WR_SECT(dev, imap_blk0_nr + i);
			break;
		}

		return inode_nr;
	}

	/* no free bit in imap */
	panic("inode-map is probably full.\n");

	return 0;
}
```

怎么看懂的？没有什么奇妙的方法，盯着代码看，猜测它的意思。

为什么花这么多时间？还浏览了很多其他代码，只用了一部分时间聚焦在这个函数。

### 2021-05-29 11:39

再看`PRIVATE int alloc_imap_bit(int dev)`，耗时22分。

注意力不集中，有问题理解不了。

1. fsbuf写入硬盘的细节。
2. 怎么通过ipc读写硬盘。

### 2021-05-29 16:27

理解下面的代码，受阻。耗时2小时6分 。

```c
/*****************************************************************************
 *                                alloc_smap_bit
 *****************************************************************************/
/**
 * Allocate a bit in sector-map.
 * 
 * @param dev  In which device the sector-map is located.
 * @param nr_sects_to_alloc  How many sectors are allocated.
 * 
 * @return  The 1st sector nr allocated.
 *****************************************************************************/
PRIVATE int alloc_smap_bit(int dev, int nr_sects_to_alloc)
{
	/* int nr_sects_to_alloc = NR_DEFAULT_FILE_SECTS; */

	int i; /* sector index */
	int j; /* byte index */
	int k; /* bit index */

	struct super_block * sb = get_super_block(dev);

	int smap_blk0_nr = 1 + 1 + sb->nr_imap_sects;
	int free_sect_nr = 0;

	for (i = 0; i < sb->nr_smap_sects; i++) { /* smap_blk0_nr + i :
						     current sect nr. */
		RD_SECT(dev, smap_blk0_nr + i);

		/* byte offset in current sect */
		for (j = 0; j < SECTOR_SIZE && nr_sects_to_alloc > 0; j++) {
			k = 0;
			if (!free_sect_nr) {
				/* loop until a free bit is found */
				if (fsbuf[j] == 0xFF) continue;
				for (; ((fsbuf[j] >> k) & 1) != 0; k++) {}
				free_sect_nr = (i * SECTOR_SIZE + j) * 8 +
					k - 1 + sb->n_1st_sect;
			}

			for (; k < 8; k++) { /* repeat till enough bits are set */
				assert(((fsbuf[j] >> k) & 1) == 0);
				fsbuf[j] |= (1 << k);
				if (--nr_sects_to_alloc == 0)
					break;
			}
		}

		if (free_sect_nr) /* free bit found, write the bits to smap */
			WR_SECT(dev, smap_blk0_nr + i);

		if (nr_sects_to_alloc == 0)
			break;
	}

	assert(nr_sects_to_alloc == 0);

	return free_sect_nr;
}
```

`free_sect_nr = (i * SECTOR_SIZE + j) * 8 +k - 1 + sb->n_1st_sect;`中，`- 1 + sb->n_1st_sect`是什么意思？

我做了什么？

1.  盯着代码看，无思路，无推理，无猜想。反而，生出其他问题，在不应该有疑问的地方产生疑问。
2. 看《操作系统真相还原》，无收获，也没有看懂作者在说什么。
   1. 两本书的文件系统设计方案差别非常大，不可能从对方那里获得收益。
   2. 不如，索性看明白郑上神的文件系统方案。
3. 于上神讲解得不详细，也没有说为啥要那样设计。代码和文字讲解结合起来，才是完整的文件系统方案。

### 2021-05-29 21:18

仍然看不懂上面的小结中的疑惑。耗时26分，总耗时大约1天，无收获。先跳过去吧。

### 2021-05-29 21:48

看`get_inode`函数，看不懂。耗时14分。

这个函数在做什么？

已经分配了inode，并且设置了inode-map中对应的bit，又从inode-table中取出一个inode，是啥意思？

### 2021-05-30 05:21

理解了`alloc_smap_bit`函数中的`free_sect_nr = (i * SECTOR_SIZE + j) * 8 +k - 1 + sb->n_1st_sect;`。

耗时32分。碰碰运气，再想想这个问题，突然就想明白了。仍然理解得不透彻，具体说，是不能解释得很清楚。主要时间消耗在写下面的解释。

`- 1 + sb->n_1st_sect`，应该怎么理解？

1. 使用sector_map时，初始位置并不是第0个bit。
   1. 使用sector_map时，初始位置是第0个bit。
   2. 但是，free_sect_nr 是什么？它不是第一个值为0的bit在sector_map中的索引。
   3. 那么，free_sect_nr 是什么？
   4. 它是，文件的第一个扇区的LBA地址。重点是，LBA地址。
   5. sb->n_1st_sect 是包括根目录在内的数据区域前面所有扇区的数量，是偏移量。
      1. 偏移量是`sb->n_1st_sect `的扇区的LBA地址是多少？
      2. 是，`sb->n_1st_sect`。
      3. 例如，偏移量是4，LBA地址是4。
2. ~~而是`sb->n_1st_sect-1`。~~
   1. ~~为什么不是`sb->n_1st_sect`？~~
   2. ~~因为，`sb->n_1st_sect`不是索引，而是数量。这是最最重要的一句话。~~
      1. ~~从何得知，`sb->n_1st_sect`不是索引，而是数量？~~
      2. ~~从超级块的每个成员之间的关系计算，可以知道，`sb->n_1st_sect`不是索引，而是数量。~~
   3. ~~例如，`sb->n_1st_sect`是3，意味着，从sector_map的第一个bit开始（初始计数器是1），到第2个bit，都是不能使用的。~~
   4. ~~什么叫不能使用？使用sector_map中的bit时，跳过第1个bit、第2个bit，第3个bit才是能够使用的有效bit位。~~
   5. ~~上面的第1个、第2个、第3个，是数量，不是索引。这是个非常非常常规的问题，常规到在平时、根本不需要解释。~~
   6. ~~数量“第1个、第2个、第3个”转换成初始值是0的数组式索引，应该是“第0个、第1个、第2个”。~~
   7. ~~能够使用的第一个有效bit是“第3个”（数量），转换成数组式索引，应该是“第2个”（初始值是0）。~~
   8. ~~这就是`sb->n_1st_sect-1`而不是`sb->n_1st_sect`的原因。~~

### 2021-05-30 06:07

理解`alloc_imap_bit`和`alloc_smap_bit`。耗时47分 。

下面的图很重要。

![image-20210530061014094](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/image-20210530061014094.png)

`i-node map`用映射表的方式记录`inode_array`的使用情况，inode_array的哪个元素被使用了，哪个元素没有被使用。

`sector map`用映射表的方式记录本分区（主分区或逻辑分区）的所有扇区的使用情况。

正因为如此，创建文件时，分配`sector map`时，`sector map`的第一个有效bit的索引是`super_block.nr_1st_sector-1`。

在图中，创建文件时，能使用的扇区的第一个有效扇区的初始位置是从root开始，而不是从boot sector开始。如此，上面的那句话，就是理所当然的了。

`inode_array`中，哪个元素能被使用，初始元素的索引是0。

整个分区中，哪个扇区能被使用，初始元素的索引是`super_block.nr_1st_sector-1`。

这种差别，在`alloc_imap_bit`和`alloc_smap_bit`记录bit使用情况时反映出来。

### 2021-05-30 06:52

理解get_inode中的`int blk_nr = 1 + 1 + sb->nr_imap_sects + sb->nr_smap_sects + ((num - 1) / (SECTOR_SIZE / INODE_SIZE));`。

耗时2个小时左右，有效时间大概半个小时。

有重大突破。

1. 知道了一个特例，num = 1。
   1. num是inode-map中的bit的序号（初始序号是0）。
   2. num = 1，表示inode-array中的第一个元素，这个元素在inode-array中的索引是0。
   3. inode-map的第0个bit是保留位，但是，inode-array的第0个元素是有效元素。
   4. inodde-array的索引N，inode-map的bit的序号M，N = M - 1。

在发现这个特例前，盯着代码看，没有任何思路，心算，想自圆其说。这是在浪费时间。

后来，我发现了上面的特例，确定了num是什么，在这个前提下，正确理解了blk_nr的计算方式。

挺灰心的。文件系统代码这么多，先理解，再记住，要耗费多少时间啊。

### 2021-05-30 13:25

躺在床上看代码，理解下面的内容：

1. get_inode中的`int blk_nr = 1 + 1 + sb->nr_imap_sects + sb->nr_smap_sects + ((num - 1) / (SECTOR_SIZE / INODE_SIZE));`。
2. `PRIVATE void hd_rdwt(MESSAGE * p)`中的`u32 sect_nr = (u32)(pos >> SECTOR_SIZE_SHIFT); `。

耗费49分钟。时间消耗基本正常。主要时间消耗在写笔记。

### 2021-05-30 15:01

理解sync_inode。耗时26分钟。

和get_inode高度相似，我依然需要重新理解。很担心，过几天，再看这些代码，我又需要重新理解。只能掌握到这个程度，怎么能自己实现文件系统啊？

我把笔记写在了代码中。

### 2021-05-30 15:19

理解`get_super_block`和`read_super_block`。耗时17分钟。

比较简单。跳过了IPC。

### 2021-05-30 16:35

理解mkfs。耗时51分。

收获：

1. 忘记记录了。以后不允许这样。每次学习，都按流程记录好。

### 2021-05-31 12:47

理解mkfs和get_inode。耗时4个小时左右，有效时间大概1个多小时。

mkfs中最难理解的是初始化sector-map。对我来说，难点是：

1. LBA寻址方式，LBA的初始值是0，而不是1。单独把这个知识点拿出来说，毫无难度。但和其他知识组合起来，有时会阻碍我理解全部过程。
2. 计算机中的索引从0开始、偏移量、第1个扇区、第1个bit等，若是单独拎出来，也无难度。在一些场景中，很影响我理解代码。
3. 在于上神的文件系统中，RW_SECT读写数据的单位是扇区。把一个扇区的数据读取到fsbuf指向的内存中后，fsbuf仍指向这片内存初始位置，而不是末尾位置。
   1. 例如，读取sector-map区域的第一个扇区后，fsbuf指向的是sector-map区域的第一个扇区的开始位置，而不是第一个扇区的最后位置。需要设置sector-map的第N个bit的值，只需使用 fsbuf + N  。N的初始值是0。
4. sector-map记录本分区数据区域的扇区使用情况，一个bit记录一个扇区的使用情况。
   1. sector-map并没有记录本分区的所有扇区的使用情况。
   2. 因为在使用RW_SECT读写硬盘时，使用的LBA地址的初始值是本分区的第一个扇区，让我误以为sector-map记录本分区所有扇区的使用情况。
   3. 初始化文件系统时，初始化sector-map仅仅设置sector-map的前(1+根目录占用的bit数)个bit，所以，sector-map仅仅记录本分区数据区域的所有扇区的使用情况。
5. 初始sector-map的方式：
   1. 第一个扇区，精细化处理。
   2. 后面的扇区，以扇区为单位初始化，把每个bit设置成0。
6. 遇到难题，不要心算。我的心算能力不强，用笔算。本次3个小时耗费在“盯着代码看、心算”。心算，实际是没有思考或瞎思考。
   1. 啥叫思考？有条理、有根据、有推理、有猜想、有论证，而不是什么也没有想。

一个疑问：LBA的初始值是本分区的第0号扇区还是整个硬盘的第0号扇区？

### 2021-05-31 14:14

理解fd_desc、filp在文件创建中的作用，看懂了许多。耗时43分。没有多少难点。多数难点已经在前面被理解了。

不过，又遇到理解不了的地方：

1. alloc_smap_bit中分配free bit的索引的计算方法。之前已经理解了，我发现我又忘记了怎么理解它。
2. 一段文字。

![image-20210531141716396](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/image-20210531141716396.png)

### 2021-05-31 16:49

阅读`do_rdwt`，看不懂细节。耗时27分钟。

### 2021-05-31 18:25

阅读do_rdwt，看懂了许多。耗时23分钟。

### 2021-06-01 07:33

阅读do_rdwt，初步理解了读文件中的下面的代码。

```c
for (i = rw_sect_min; i <= rw_sect_max; i += chunk) {
			/* read/write this amount of bytes every time */
			int bytes = min(bytes_left, chunk * SECTOR_SIZE - off);
  		// chunk 的值，大于等于1
  		// rw_sect_min 的值大于等于pin->i_start_sect。认识到这一点，很重要。i的初始值是rw_sect_min。我之前误认为 i 的初始值等于0。
  		// 又有疑点。一个文件的i_start_sect是5（初始值是0），读取的是第6个扇区（初始值是1）。在所有的地方，初始值相同，就不会乱套。
  		// 此处有疑点，可能是受到inode-map和inode-array的索引不一致的影响。
  		// inode-map中的第0个bit是保留位，inode-map的第1个bit，对应inode-array中的第1个元素。
  		// 总之，
			rw_sector(DEV_READ,
				  pin->i_dev,
				  i * SECTOR_SIZE,
				  chunk * SECTOR_SIZE,
				  TASK_FS,
				  fsbuf);

			if (fs_msg.type == READ) {
				phys_copy((void*)va2la(src, buf + bytes_rw),
					  (void*)va2la(TASK_FS, fsbuf + off),
					  bytes);
			}
			else {	/* WRITE */
				phys_copy((void*)va2la(TASK_FS, fsbuf + off),
					  (void*)va2la(src, buf + bytes_rw),
					  bytes);

				rw_sector(DEV_WRITE,
					  pin->i_dev,
					  i * SECTOR_SIZE,
					  chunk * SECTOR_SIZE,
					  TASK_FS,
					  fsbuf);
			}
			off = 0;
			bytes_rw += bytes;
			pcaller->filp[fd]->fd_pos += bytes;
			bytes_left -= bytes;
		}
```



又花了20分钟，理解了上面的函数中的READ部分。

又花了36分钟，理解了上面函数中的大部分，除开下面的：

```c
int imode = pin->i_mode & I_TYPE_MASK;

	if (imode == I_CHAR_SPECIAL) {
		int t = fs_msg.type == READ ? DEV_READ : DEV_WRITE;
		fs_msg.type = t;

		int dev = pin->i_start_sect;
		assert(MAJOR(dev) == 4);

		fs_msg.DEVICE	= MINOR(dev);
		fs_msg.BUF	= buf;
		fs_msg.CNT	= len;
		fs_msg.PROC_NR	= src;
		assert(dd_map[MAJOR(dev)].driver_nr != INVALID_DRIVER);
		send_recv(BOTH, dd_map[MAJOR(dev)].driver_nr, &fs_msg);
		assert(fs_msg.CNT == len);

		return fs_msg.CNT;
	}
```

inode的i_mode是什么？

### 2021-06-01 11:49

思考do_unlink的设置inode-map部分。基本写出来了。耗时39分。

### 2021-06-01 15:17

阅读do_unlink的设置sector-map、清空inode-array部分。看不懂。耗时18分，看不懂。

### 2021-06-01 15:56

阅读do_unlink的设置sector-map、清空inode-array部分。看不懂。耗时46分，看不懂。

使用了断点调试，代入具体数字来理解，仍然理解不了。

十分担忧。这些需要多步计算才能明白的代码，即使费力理解了，又能怎样呢？下次遇到，可能又会忘记。

### 2021-06-01 16:29

阅读do_unlink的设置sector-map、情况inode-array部分。看不懂。耗时33分，有点收获。

认识到：每个文件最多使用2048个扇区。每个文件在sector-map中占用的bit不足512*8(一个扇区的bit数量)个。

中途，理解不了，有点烦，看了v2ex和公众号。

### 2021-06-01 20:42

阅读do_unlink的设置sector-map，有了一些新理解。耗时45分。

进展缓慢。有什么办法呢？只能盯着代码看，举例子，也举不出更能发现问题的例子；断点调试，也不能帮助我发现问题。

一点点收获：对目标文件在sector-map中的数据的第一个字节和最后一个字节需要特殊处理，其他字节直接设置成0。

### 2021-06-02 20:41

阅读do_unlink的设置sector-map，耗时 。

理解的内容是：

```c
/**************************/
	/* free the bits in s-map */
	/**************************/
	/*
	 *           bit_idx: bit idx in the entire i-map
	 *     ... ____|____
	 *                  \        .-- byte_cnt: how many bytes between
	 *                   \      |              the first and last byte
	 *        +-+-+-+-+-+-+-+-+ V +-+-+-+-+-+-+-+-+
	 *    ... | | | | | |*|*|*|...|*|*|*|*| | | | |
	 *        +-+-+-+-+-+-+-+-+   +-+-+-+-+-+-+-+-+
	 *         0 1 2 3 4 5 6 7     0 1 2 3 4 5 6 7
	 *  ...__/
	 *      byte_idx: byte idx in the entire i-map
	 */
	// bit_idx 应该怎么理解？
	// 第一种理解，从n_1st_sect到i_start_sect包含的扇区数量。
	// 第二种理解，bit_idx是sector-map的索引，sector-map的索引的第0位是保留的，比数据区的扇区的序号大1。
	// 假如 n_1st_sect = 0，i_start_sect = 1，那么，bit_idx = 2。
	// 具体情况是：在sector-map中，第0号是保留位，第1号是n_1st_sect，第2号是i_start_sect。
	// 像这种corner case，让我很烦。实在说不清为啥如此，那就这么说吧，这是用归纳法归纳出来的计算公式。
	// n_1st_sect = 1，i_start_sect = 3， bit_idx = 4。
	// 上面的理解非常混乱。
	// pin->i_start_sect、 sb->n_1st_sect 都是 LBA 地址。
	// 二者的差，是两个刻度之间的偏移量。
	// 但是，我需要的，不是这两个刻度之间的偏移量，而是i_start_sect在sector-map中对应的bit和sector-map的第0号bit的偏移量。
	// 目标偏移量 = i_start_sect - n_1st_sect + 1。n_1st_sect在sector-map中对应第1号bit。
	bit_idx  = pin->i_start_sect - sb->n_1st_sect + 1;
	byte_idx = bit_idx / 8;
	int bits_left = pin->i_nr_sects;
	// bits_left 是数量还是初始值是0的索引？是数量。
	// byte_cnt 是数量还是初始值是0的索引？
	// 当 bits_left = 1 时，bit_idx = 1，byte_cnt = 0。
	int byte_cnt = (bits_left - (8 - (bit_idx % 8))) / 8;

	// s是什么？它是目标文件占用的扇区在sector-map中的索引。
	// 再具体一些，读取第s号扇区（初始值是0），所读取到一个扇区的数据记录目标文件在数据区域
	// 的最开始的512*8个扇区（初始值是0）的使用情况。
	/* current sector nr. */
	int s = 2  /* 2: bootsect + superblk */
		+ sb->nr_imap_sects + byte_idx / SECTOR_SIZE;

	RD_SECT(pin->i_dev, s);

	int i;
	// 为什么需要特殊处理the first byte？
	// 因为第一个字节的前N个bit记录了目标文件的sector的使用情况，第一个字节的(8-N)和目标文件毫无关系。
	// bit_idx 是目标文件在数据区占用的扇区的数量。
	// bit_idx % 8 是把bit数量换算成字节数量后不足一个字节的bit数量。
	// byte_idx 是把bit数量换算成字节数量。
	// byte_idx % SECTOR_SIZE 是把字节数量换算成扇区数量后不足一个扇区的剩余的字节数量。
	// i < 8，为什么不是 i <= 8？这类问题，举例理解最好。
	// 当 bit_idx % 8 = 0 时，i的范围是[0,7]，正好8个，一个字节。
	// 当 bit_idx % 8 = 1 时，i的范围是[1, 7]，正好7个，加上前面的第0个，一个字节。
	// 现在回答为什么不是 i <= 8？因为 i <= 8时，处理的bit数加上前面未处理的字节数，总计9个bit。
	// 而我们只应该处理8个bit，也就是，我们只应该处理1个字节。
	/* clear the first byte */
	for (i = bit_idx % 8; (i < 8) && bits_left; i++,bits_left--) {
		assert((fsbuf[byte_idx % SECTOR_SIZE] >> i & 1) == 1);
		fsbuf[byte_idx % SECTOR_SIZE] &= ~(1 << i);
	}

	// the second to last 倒数第二
	// 从第二个到倒数第二个字节，全部设置成0。
	/* clear bytes from the second byte to the second to last */
	int k;
	i = (byte_idx % SECTOR_SIZE) + 1;	/* the second byte */
	// 为什么不是 k <= byte_cnt ？
	// 仍然举例子。
	// 当 byte_cnt = 1 时，这个循环执行一次，仍有未处理完的bit。
	for (k = 0; k < byte_cnt; k++,i++,bits_left-=8) {
		if (i == SECTOR_SIZE) {	
			i = 0;
			WR_SECT(pin->i_dev, s);
			RD_SECT(pin->i_dev, ++s);
		}
		assert(fsbuf[i] == 0xFF);
		fsbuf[i] = 0;
	}

	// 未处理完的bit，在最后一个字节处理。
	// 最后一个字节，又需要特殊处理。为什么？
	/* clear the last byte */
	if (i == SECTOR_SIZE) {
		i = 0;
		WR_SECT(pin->i_dev, s);
		RD_SECT(pin->i_dev, ++s);
	}
	unsigned char mask = ~((unsigned char)(~0) << bits_left);
	assert((fsbuf[i] & mask) == mask);
	fsbuf[i] &= (~0) << bits_left;
	WR_SECT(pin->i_dev, s);
```

时间主要消耗在上面的代码中的第一段注释。

举例子理解这样的代码，是个非常好的方法。

阻碍我的因素：

1. 刻度尺子上的两个刻度之间的关系、文件系统中sector-map和文件中扇区的LBA地址间的关系。
2. 违背了“思维定势”。例如，`int byte_cnt = (bits_left - (8 - (bit_idx % 8))) / 8`的一个具体实例`int byte_cnt = -7 / 8`。我觉得公式中出现了`-7`是不正常的。这也是正常的。公式中出现了负数，计算结果仍然正确。

经验是，多举例子理解代码。盯着代码看，没有作用。

很担忧，在这种没有通用性的知识点上耗费这么多时间，效率太低，回报太少，太让我沮丧、灰心。

### 2021-06-02 23:18

终于理解了link.c中的全部内容，后续再看有没疑问。耗费了2个小时3分钟。

第一时间消耗点是：strip_path中的 `struct inode** ppinode`。

这是C语言知识点。我以前理解了，再看到时，又花了很多时间理解。即使不理解也没有关系，我能够正确地使用指针。我不能从汇编的角度去解释变量、指针、指向指针的指针。

第二时间消耗点是：清空根目录。不懂细节，也能从整理体猜测代码的意思。从整体上明白了代码要做什么，又能反过来理解代码细节。

### 2021-06-03 07:45

阅读文件系统和tty。没看懂二者的关系，更没看懂代码细节。耗时1小时26分。

两个时间消耗点：

1. 理清代码流程。仍没有理清。阻碍点是IPC。
2. 不知道tty接入文件系统后，我在操作系统的使用中能看到什么。

要解决两个问题：

1. 弄清流程。
2. 让tty能运行。
   1. 键盘驱动正常吗？
   2. 能切换终端吗？

### 2021-06-03 09:14

阅读文件系统和tty。无收获。耗时。

做了两件事：

1. 综合使用xargs、awk、grep等查找进程并且杀死这个进程。
2. 验证了j版代码的时钟中断、键盘中断都在正常运行。

可是，为啥在键盘输入数据没却不在显示器打印出来呢？看来，难题是解决这个问题了。

### 2021-06-03 15:50

理清`TestB中的int fd_stdin  = open(tty_name, O_RDWR);`流程，没有断点验证，是合理猜测的结果。耗费时间50分。

不难，但是要仔细。

我用多个编辑器看代码，在物理机和虚拟机之间切换。使用vim、sublime Text4。

能使用vim的高级功能，差不多能完全脱离那些界面IDE了。

### 2021-06-03 16:47

梳理`int r = read(fd_stdin, rdbuf, 70);`的执行流程，失败了！耗费时间52分。

为什么失败？按照我梳理出来的流程，流程中出现死循环，没有搞清楚tty什么时候等待输入。

### 2021-06-03 17:55

梳理`int r = read(fd_stdin, rdbuf, 70);`的执行流程，失败了！耗费时间1小时20分。

我使用bochs汇编中断点，失败了。

时间全部消耗在设置断点、查看数据。

为啥？太复杂了！cs在变化，我连确定正确的断点地址都做不到。

1. 目前，cs的所有值表示的段地址都是0，直接在反汇编出来的的代码对应的内存地址设置断点就行。
2. 遇到新问题。sys_sendrec在open之前被多次调用。若在这里设置断点，需要运行很多次才能捕捉到断点。
3. 手工捕捉断点，实在太麻烦，看能不能用代码来捕捉信息。

这个方法，太复杂了。即使要在汇编代码中断点，也需要先想好断点方案。

原始的调试方法，已经耗费了太多太多时间。我决定，尝试记录日志来查看执行流程。

### 2021-06-03 23:28

于上神的i版代码是能够运行的。只是我没有切换到正确的终端。耗费时间30分。

切换方法：

1. option + Fn + F2
2. f2

### 2021-06-04 10:47

阅读tty和文件系统代码，有一些进展，耗费时间 2个小时22分。

所做的事分为几个阶段：

1. 验证于上神的代码能不能运行。耗时很少。
2. 梳理open终端的执行流程。
   1. 比较费时间。猜测 + 断点验证关键数值 + 笔算，弄清了流程。
   2. 弄清楚open终端的流程后，很快看明白了read终端、write终端的流程。
3. write终端的流程。
   1. 在哪里解除阻塞？书上的解说，加上grep搜索msg的type，让我看明白了解除阻塞的代码。
4. read终端流程，我仍有疑问。
   1. 在哪里解除阻塞？
   2. 我只看到了解除write的阻塞的代码，没有看到解除read的阻塞的代码。

为什么耗费这么多时间？

1. 我觉得，浪费了不少时间，思路不顺畅的时候，我没有条理清晰地考虑问题，好像在走神，或者根据模糊的想法乱看代码。
2. 看代码，确实非常耗费时间。即使只看一个汉字，也能看上一个小时。做任何事情，只要反复在上面耗，都非常耗费时间。
3. 不靠谱的心算。

### 2021-06-04 12:07

阅读tty和文件系统代码，明白了解除read 终端的代码。耗费41分，实际只耗费了大概20分钟。

在哪里解除read终端的阻塞？

```c
PUBLIC void task_tty()
{
	// some code

	while (1) {
		for (tty = TTY_FIRST; tty < TTY_END; tty++) {
			do {
				tty_dev_read(tty);
        // 解除阻塞的代码在tty_dev_write中。
        // 两种情况下会解除阻塞：1. 读取到换行符；2. 读取到的数据的长度足够。
				tty_dev_write(tty);
			} while (tty->ibuf_cnt);
		}
	// some code
	}
}
```

为什么能看明白？

1. 休息了一会，碰碰运气再看，想到了新思路。全文检索tty_dev_write，发现了上面的代码。
2. 理解上面的代码。发现，它能解除read终端的阻塞。

休息 + 猜想 + 验证，解决了问题。

### 2021-06-04 16:32

阅读下面的代码。耗时1个小时36分。

```c
// /Users/cg/data/code/os/yy-os/osfs09/i/fs/main.c 中的 get_inode
struct super_block * sb = get_super_block(dev);
	int blk_nr = 1 + 1 + sb->nr_imap_sects + sb->nr_smap_sects +
		((num - 1) / (SECTOR_SIZE / INODE_SIZE));
	RD_SECT(dev, blk_nr);
	struct inode * pinode =
		(struct inode*)((u8*)fsbuf +
				((num - 1 ) % (SECTOR_SIZE / INODE_SIZE))
				 * INODE_SIZE);
```

很烦这种问题。之前理解了，再看到时，又耗费这么多时间。

这是什么问题？根据inode在inode-map中的索引计算inode的LBA地址，计算inode的初始地址，获取inode的数据。

为啥耗费这么多时间？理解不了的时候，看了其他凤凰网等无用资讯。理解后，写笔记。

最关键的笔记：num - 1, 确实是bit数量，不过，在这里，应该理解为inode的数量。

`int blk_nr = 1 + 1 + sb->nr_imap_sects + sb->nr_smap_sects +((num - 1) / (SECTOR_SIZE / INODE_SIZE));`

修改成

`int blk_nr = 1 + 1 + sb->nr_imap_sects + sb->nr_smap_sects +((num - 1) * INODE_SIZE / SECTOR_SIZE);`

更好理解。

### 2021-06-04 18:12

理解了get_inode。耗时2个小时25分。我很不满意这种效率。

时间消耗在：

1. 对inode_table的使用。
   1. inode_table是内存中缓存inode的区域，避免每次都读硬盘。
   2. inode_table只缓存正在使用的文件的inode，没有被任何进程使用的文件的inode不会被缓存在inode_table中。
   3. 我的成见“inode_table缓存了所有文件的inode"让我在错误中徘徊了很久很久。我不知道为啥会有这样的成见。
2. 从硬盘中读取的inode的默认值。
   1. 从硬盘中读取的inode可能有默认值，也可能没有默认值。
   2. 默认值是在new_inode中赋予的，然后保存到硬盘中。
3. 指针的使用，让从get_inode获取的inode成为一个全局变量。对inode的任何修改实质是对inode_table中inode的修改。我觉得这种用法很危险。

为啥会花这么多时间？

1. 理解不了，我就不停看各种代码，无头苍蝇一般。
2. 我回忆不起来我所做的每件事，也就是说，我不记得我为啥会花这么多时间。
   1. 要更加细致地记录我做的事情。
   2. 理解代码，非常花时间。

### 2021-06-04 21:03

粗略看了一次tty和read、open的流程，对我来说，纠结点仍是”根据indoe在inode-array“中的索引来计算目标inode所在的扇区的地址等和索引、初始值这类有关 尝试。

耗时36分钟。

又耗时24分钟读kernel/main.c中TestB的代码。

基本读懂全部文件系统代码后，我忽然不知道接下来我应该做什么。我挑了几段代码，发现并不是我想象中那样全都懂。

还有好大的工作量。整个文件系统，不算简单，各种进程互相调用。我该怎么做？

1. 凭记忆写出文件系统。
2. 读一块代码，就通过写文章复述。
3. 完全自己设计文件系统并实现。
4. 先把日志功能做出来。理清代码执行流程，有时真不容易。

编译原理：

中间语言：https://zh.wikipedia.org/wiki/%E4%B8%AD%E9%96%93%E8%AA%9E%E8%A8%80

### 2021-06-16 16:58

重读本笔记的v5-v13，看不懂很多内容。

复习，要复习，要尽早复习。

我其实根本就没有复习过。不能过目不忘，又想记住，不复习怎么行呢？

好不容理解的内容，不复习，过了一个月，这些内容又变成陌生知识了，又要耗费大量时间。

### 2021-06-16 22:18

重读本笔记v13~~疑问。大部分看不懂。最多时间消耗在疑问中的那个简单的C语句。耗时2个小时23分。

下次，要避免在这种小问题、可以暂时搁置的问题上耗费很多时间。·

### 2021-06-16 23:07

读本笔记的所有小结。能看懂一些内容。耗时55分。

坚持写详细的小结，坚持复习笔记。

半个月前、一个月前的笔记，我已经看不懂了。

写了注释的代码在：/home/cg/yuyuan-os/osfs09/e。

### 2021-06-16 23:56

再次验证“疑问》段错误"，无收获。

疑问：

1. 使用`char str[20] = "hello";*str='A';`，对应汇编中的应该是str的内存地址不可访问（实际上不是）。见下面gdb调试截图：

   1. ```shell
      $3 = (char (*)[10]) 0xffffd36e
      (gdb) s
      5		*str = 'A';
      (gdb) p &str
      $4 = (char (*)[10]) 0xffffd36e
      (gdb) x /1wc 0xffffd36e
      0xffffd36e:	104 'h'
      (gdb)
      0xffffd372:	111 'o'
      (gdb) x /1wc 0xffffd36e
      0xffffd36e:	104 'h'
      (gdb) x /1wx 0x6c6c6568
      0x6c6c6568:	Cannot access memory at address 0x6c6c6568
      ```

   2. 使用`char *str = "hello";*str='A';`，对应汇编中的应该是str的内存地址可访问。

   3. `*str`的数据类型是char，第1点种的情况，能赋值；第2点种的情况，不能赋值。为什么？

### 2021-06-17 12:12

重看本笔记v1~~~v11，看不懂v11。

先看懂笔记，再重看代码。在笔记中没有看懂的问题，在代码中应该也看不懂。

笔记有比较大的价值，值得看。

### 2021-06-17 14:59

重看v11。理解了次设备号。耗时1个小时28分。又在分区表的base计算上耗费20分钟。

仍有疑问：

```c
// 设备的次设备号不能重复，但是，这里的dev_nr却包含了[0,9]，如何理解？
int dev_nr = nr_1st_sub + i;/* 0~15/16~31/32~47/48~63 */
```

我很纠结分区表的位置。它在逻辑分区中吗？在子扩展分区中吗？在哪里，不知道也不影响理解代码。

通过分区表，能找到本扩展分区中的逻辑分区的偏移量，还能找到下一个子扩展分区的偏移量。

为什么慢？

1. 大量的宏，需要跳来跳去找出数值。计算出结果，把具体数值代入宏表达式中能让我集中精力看表达式的意思。
2. 不理解，但不知道不理解什么。逐渐、自发地弄明白了我的困惑。
3. 思路停滞时，不要心算。

> 时间久了，又会忘记，而且还会看不懂这些笔记。

### 2021-06-17 17:41

纠结v13中的get_inode中的这段代码。心算。耗时45+20分。这是非常低效的表现。这是一个出现了很多次的常识。有效时间只有四五分钟。其他时间，我在走神，

自由散漫无章法而又焦虑地想理解这个问题。为啥焦虑？为这样的问题浪费时间。

早点举例子来理解，根本就不会浪费这么多宝贵的时间！

```shell
// file：/home/cg/yuyuan-os/osfs10/e/fs/main.c#get_inode
int blk_nr = 1 + 1 + sb->nr_imap_sects + sb->nr_smap_sects +
		((num - 1) / (SECTOR_SIZE / INODE_SIZE));
	RD_SECT(dev, blk_nr);
```

1. num是inode-map中的索引。inode-map的索引的0是保留位，不记录任何inode-array的元素的使用情况。
2. Inode-array的第0个元素，使用。inode-map中的第1个bit，对应inode-array中的第0个元素。依此类推，num对应inode-array中的第num-1个元素。
3. RED_SECT的第二个参数blk_nr是目标数据的开始位置，即，从哪里开始读取目标数据。
4. 省略掉前面的，只分析在inode-array中的位置。
5. 读取第0个数据，从第0个元素的开头读取。
6. 读取第1个数据，从第1个元素的开头读取。
7. 读取第2个数据，从第2个元素的开头读取。
8. 只有初始序号是0，才能满足上面的要求。
9. 当初始序号是1，即，第一个数据的序号是1时，如果读取位置是1，会怎样？将会从第1个元素的末尾读取。从第1个元素的末尾读取1个数据，读取的是第2个数据。这不是我期望读取的。

我不满意上面的表述。能用更少的语言说清楚吗？

要读取第N个元素，必须从第N-1个元素的末尾即第N个元素的开头读取。

> 很烦这种小问题。
>
> 不该在这种小问题上浪费一个小时。

配合图来表述会很好说。

| A    | B    | C    |
| ---- | ---- | ---- |

1. 要读取A（第1个）数据，读取位置应该是0*一个数据的长度。
2. 要读取B（第2个）数据，读取位置应该是1*一个数据的长度。
3. 要读取C（第3个）数据，读取位置应该是2*一个数据的长度。
4. 非常容易看出来，要读取第H数据（第N个），读取位置应该是(N-1)*一个数据的长度。
5. 又很容易看出来，若数据的初始编号是1，读取目标数据的读取位置序号应该是N-1；若数据的初始编号是0，读取目标数据的读取位置序号应该是N；

> 再也不想在这种常识问题上耗费时间了。我宁可死记硬背。

### 2021-06-17 20:30

看本笔记的v12、v13和其他。耗时39分钟。

笑死人了。自己写的笔记，放久了，自己都看不懂。

1. 可能是因为没有记录上下文。
2. 可能是跟不上当时的思路。

### 2021-06-17 21:03

看get_part_table、partition函数。还算顺利。耗时18分钟。

不想看了。不知道哪天才能写完操作系统。

### 2021-06-17 23:05

看《一个操作系统的实现》文件系统。

不知道从何看起了。

直接看代码？千头万绪，从哪个文件开始看？

还算跟着书的步骤看代码，最合适。

这两天，一直在重复本笔记，效果很差。竟然被自己一个月前的笔记难倒了。好笑不？

今天做了啥？

1. 看完了内存管理的笔记。比较顺利。因为，内容少，四五天前写的。
2. 文件系统笔记。纠结于分区表、初始序号是0还是1等问题。

进度太慢，心急如焚。

### 2021-06-18 17:17

![image-20210618171756157](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/image-20210618171756157.png)

看不懂这张图和对它的解释。

它出自《一个操作系统的实现》9.3硬盘参数。

耗时42分。

看不懂。图文对照着看，仍然感觉没有看懂。不想看。又接着看。就这样，42分钟过去了。又耗时若干分。总计1小时13分。

哪里看不懂？我懒得记录。

1. 第一种情况，HD Driver要操作硬盘时，硬盘正在忙，不能执行HD Driver的操作。
2. 第二种情况，HD Driver要操作硬盘时，硬盘是空闲的，能够立刻执行HD Driver的操作。

孤立地想这个问题，不会得到答案。

也许，这个问题对我而言是陌生领域，我的猜想是错误的。搁置。