---
description: 主要是笔记。
---

# 《自己动手写操作系统》笔记

/Users/cg/MYDOSBox/asm2/chapter1/a/boot.bin
/Users/cg/MYDOSBox/asm2/chapter2/linux/boot.bin
/Users/cg/MYDOSBox/asm2/chapter3/a/boot.bin
/Users/cg/MYDOSBox/asm2/chapter4/c/boot.bin
/Users/cg/data/code/wheel/asm2/chapter1/a/boot.bin
/Users/cg/data/code/wheel/asm2/chapter2/linux/boot.bin
/Users/cg/data/code/wheel/asm2/chapter3/a/boot.bin
/Users/cg/data/code/wheel/c/pegasus-os/experiment/boot.bin

## 时间消耗点

1. 软盘结构及其数据读取，文字说明、代码。耗时三四个小时。看各种不同的讲解，看代码，分心。
2. 

## 工具

bochs--can not connect to X server 问题

解决：执行 startx，然后切换到非root用户。只能在虚拟机上运行，在连接到虚拟机的命令行工具上无效。

### bochs

#### 常用命令

##### xp

```shell
# 查看内存地址为A的内存区域的数据，2是数量，w是数量单位字，x是显示数据的格式十六进制
xp /2wt 0x0000000000090145
xp /1wx 0x000000000009014D
xp /1wx 0x0000000000090155
xp /1wx 0x000000000009015D
xp /1wx 0x00000000000306d8
```



## 书本笔记

### 进程restart

![image-20210108181241749](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20210108181241749.png)



```
(0) [0x0000000306ea] 0008:00000000000306ea (unk. ctxt): lldt word ptr ss:[esp+72] ; 0f00542448
<bochs:4> s
Next at t=18317745
(0) [0x0000000306ef] 0008:00000000000306ef (unk. ctxt): lea eax, ss:[esp+72]      ; 8d442448
```



```
; ====================================================================================
;				    restart
; ====================================================================================
restart:
	mov	esp, [p_proc_ready]
	lldt	[esp + P_LDT_SEL]
	lea	eax, [esp + P_STACKTOP]			; 将一个内存地址直接赋给目的操作数
	mov	dword [tss + TSS3_S_SP0], eax
restart_reenter:
	dec	dword [k_reenter]
	pop	gs
	pop	fs
	pop	es
	pop	ds
	popad
	add	esp, 4
	iretd
```



在这里耗费了非常多时间。我不理解，esp + P_LDT_SEL 和 P_STACKTOP 的值相同，为何，书中说，esp + P_STACKTOP 是 regs 的末地址，而 esp + P_LDT_SEL 是 ldt_sel。

![image-20210108182923588](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20210108182923588.png)





![image-20210108183124423](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20210108183124423.png)



结合 pop 和 push 的执行过程，可以知道： esp + 72 这个地址，是一个分界线，往

### 8259A

#### ICW1

![image-20210107194242621](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20210107194242621.png)



初始化连接方式和中断信号的触发方式：一片还是多片；电平还是边沿触发。

需写入主片的0X20端口和从片的0XA0端口。

### 软盘

![image-20201231163731952](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201231163731952.png)

![image-20201231163827164](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201231163827164.png)



### 硬盘分区表

#### 分区表结构表

![image-20210117175244071](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20210117175244071.png)

偏移单位是字节，不是字。我根据分区表结构的代码看出来的，`u8 boot_ind` 、 u8 start_head;` 等。

```c
struct part_ent {
	u8 boot_ind;		/**
				 * boot indicator
				 *   Bit 7 is the active partition flag,
				 *   bits 6-0 are zero (when not zero this
				 *   byte is also the drive number of the
				 *   drive to boot so the active partition
				 *   is always found on drive 80H, the first
				 *   hard disk).
				 */

	u8 start_head;		/**
				 * Starting Head
				 */
  // 其他成员结构
} PARTITION_ENTRY;
```



### 硬盘

### 遍历硬盘分区

用到了两个获取硬盘信息的命令

1. `0x20`

```c
/*****************************************************************************
 *                                get_part_table
 *****************************************************************************/
/**
 * <Ring 1> Get a partition table of a drive.
 * 
 * @param drive   Drive nr (0 for the 1st disk, 1 for the 2nd, ...)n
 * @param sect_nr The sector at which the partition table is located.
 * @param entry   Ptr to part_ent struct.
 *****************************************************************************/
PRIVATE void get_part_table(int drive, int sect_nr, struct part_ent * entry)
{
	struct hd_cmd cmd;
	cmd.features	= 0;
	cmd.count	= 1;
	cmd.lba_low	= sect_nr & 0xFF;
	cmd.lba_mid	= (sect_nr >>  8) & 0xFF;
	cmd.lba_high	= (sect_nr >> 16) & 0xFF;
	cmd.device	= MAKE_DEVICE_REG(1, /* LBA mode*/
					  drive,
					  (sect_nr >> 24) & 0xF);
	cmd.command	= ATA_READ;
	hd_cmd_out(&cmd);
	interrupt_wait();

	port_read(REG_DATA, hdbuf, SECTOR_SIZE);
	memcpy(entry,
	       hdbuf + PARTITION_TABLE_OFFSET,
	       sizeof(struct part_ent) * NR_PART_PER_DRIVE);
}


/**
 * @struct part_ent
 * @brief  Partition Entry struct.
 *
 * <b>Master Boot Record (MBR):</b>
 *   Located at offset 0x1BE in the 1st sector of a disk. MBR contains
 *   four 16-byte partition entries. Should end with 55h & AAh.
 *
 * <b>partitions in MBR:</b>
 *   A PC hard disk can contain either as many as four primary partitions,
 *   or 1-3 primaries and a single extended partition. Each of these
 *   partitions are described by a 16-byte entry in the Partition Table
 *   which is located in the Master Boot Record.
 *
 * <b>extented partition:</b>
 *   It is essentially a link list with many tricks. See
 *   http://en.wikipedia.org/wiki/Extended_boot_record for details.
 */
struct part_ent {
	u8 boot_ind;		/**
				 * boot indicator
				 *   Bit 7 is the active partition flag,
				 *   bits 6-0 are zero (when not zero this
				 *   byte is also the drive number of the
				 *   drive to boot so the active partition
				 *   is always found on drive 80H, the first
				 *   hard disk).
				 */

	u8 start_head;		/**
				 * Starting Head
				 */

	u8 start_sector;	/**
				 * Starting Sector.
				 *   Only bits 0-5 are used. Bits 6-7 are
				 *   the upper two bits for the Starting
				 *   Cylinder field.
				 */

	u8 start_cyl;		/**
				 * Starting Cylinder.
				 *   This field contains the lower 8 bits
				 *   of the cylinder value. Starting cylinder
				 *   is thus a 10-bit number, with a maximum
				 *   value of 1023.
				 */

	u8 sys_id;		/**
				 * System ID
				 * e.g.
				 *   01: FAT12
				 *   81: MINIX
				 *   83: Linux
				 */

	u8 end_head;		/**
				 * Ending Head
				 */

	u8 end_sector;		/**
				 * Ending Sector.
				 *   Only bits 0-5 are used. Bits 6-7 are
				 *   the upper two bits for the Ending
				 *    Cylinder field.
				 */

	u8 end_cyl;		/**
				 * Ending Cylinder.
				 *   This field contains the lower 8 bits
				 *   of the cylinder value. Ending cylinder
				 *   is thus a 10-bit number, with a maximum
				 *   value of 1023.
				 */

	u32 start_sect;	/**
				 * starting sector counting from
				 * 0 / Relative Sector. / start in LBA
				 */

	u32 nr_sects;		/**
				 * nr of sectors in partition
				 */

} PARTITION_ENTRY;
```



突然发现C语言中的结构也挺好用的，比对象更好用。一堆数据，二进制数据，天然就是struct，只不过有分界线而已。这块数据是这个成员变量，那块数据是那个成员变量。

看了几个二进制文件，对内存的有了很形象的认识。内存地址，不过就是在一块区域（与文件相似）的里的位置、坐标、偏移量，物理地址就是这块区域的物理地址加上偏移量。



2. `0xEC`

```c
/*****************************************************************************
 *                                hd_identify
 *****************************************************************************/
/**
 * <Ring 1> Get the disk information.
 * 
 * @param drive  Drive Nr.
 *****************************************************************************/
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
	hd_info[drive].primary[0].size = ((int)hdinfo[61] << 16) + hdinfo[60];
}
```







#### 硬盘参数

![image-20210117161529911](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20210117161529911.png)



![image-20210117161608879](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20210117161608879.png)



《一个操作系统的实现》，缺少许多细节，任何一个在作者看来小得不能再小的知识点，都能阻挡我四五个小时甚至更多。

偏移量是字，不是字节。所以，序列号的偏移是10~19,有20个字符。

还有一点，作者们都不提，那就是0xec命令返回的数据是小端法。

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
			// 打印的字节顺序和读取到的硬盘参数字节顺序相反，以字节为单位。
			s[i*2+1] = *p++;
			s[i*2] = *p++;
		}
		s[i*2] = 0;
		printl("%s: %s\n", iinfo[k].desc, s);
	}

	int capabilities = hdinfo[49];
  0x0200，第9位是0，当capabilities的第9位是1时，支持LBA。
	printl("LBA supported: %s\n",
	       (capabilities & 0x0200) ? "Yes" : "No");

	int cmd_set_supported = hdinfo[83];
  0x400，第10位是0，当cmd_set_supported的第10位是1时，支持LBA48。
	printl("LBA48 supported: %s\n",
	       (cmd_set_supported & 0x0400) ? "Yes" : "No");

	int sectors = ((int)hdinfo[61] << 16) + hdinfo[60];
  // 1000000 是十进制还是十六进制？十进制。这不是按照1mb = 1024 * 1024 byte计算的。
	printl("HD size: %dMB\n", sectors * 512 / 1000000);
}
```

小端法：低位数据在右边，高位数据在左边。例如，`1578`，用小端法表示是，`7815` 。

切分单位，不固定，可能是字节，也可能是字。例如，`int sectors = ((int)hdinfo[61] << 16) + hdinfo[60];`，切分单位是字。

这段代码，隐藏着“小端法”和“单位是字”两个条件。

这难吗？若硬盘返回数据是我设计的，我一定会知道这样读取并处理数据。可我不知道啊。

还剩下一个问题：C语言字符串复制。

不是我复制字符串的方法有问题，而是数据有问题，而我没有发现。偏移量从20开始，而不是从10开始。若从10开始，复制10位，都是空白的。这个结果，让我以为代码有问题。

```c
#include <stdio.h>

int main(int argc, char *argv[])
{
        int i, k;
        char s[64];

        struct iden_info_ascii {
                int idx;
                int len;
                char * desc;
        } iinfo[] = {{10, 20, "HD SN"}, /* Serial number in ASCII */
                     {27, 40, "HD Model"} /* Model number in ASCII */ };

        char *p2 = "@\000\242\000\000\000\020\000\000~\000\002?\000\000\000\000\000\000\000XBDH0010 1  @\000\242\000\000\000\020\000\000~\000\002?\000\000\000\000\000\000\000XBDH0010 1  43";
        char *p = (char *)&p2[20];
        for (i = 0; i < 10; i++) {
                        //printf("p = %c\n", *p);
                        // printf("s[i*2+1] = %c\n", *p);
                        s[2 *i] = *p++;
                        // printf("s[i*2] = %c\n", *p);
                        s[2 * i+1] = *p++;
                }
                printf("\n");
                // s[i*2]s[i*2] = 0; = 0;
                // printf("%s\n",  s);

        s[i] = 0;
        printf("%s\n",  s);
        return 0;
}
```



这段代码的价值：

1. 解析硬盘参数。

   1. 单位是字。

   2. 字节顺序转换。只需修改下面两句的顺序：

      ```c
       s[2 *i] = *p++;
       s[2 * i+1] = *p++;
      ```

      

2. `char *p = (char *)&p2[20];` 。不理解这句。

3. `*p++`，先获取p中存储的内存中的内存地址，然后再将内存地址增加一个单位。

   1. 

   ```c
   #include <stdio.h>
   
   int main(void)
   {
           int a[5]={1,2,3,4,5};
           int *p = a;
           printf("*p++ = %d\n", *p++);
           printf("*p++2 = %d\n", *p++);
           return 0;
   }
   ```

   2. 执行结果是：

      ```shell
      chugangdeMacBook-Pro:my-note-book cg$ ./pointer
      *p++ = 1
      *p++2 = 2
      ```





#### 机械硬盘

![image-20201231170645288](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201231170645288.png)

![image-20201231171607871](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201231171607871.png)

### 获取内存

![image-20201230162917205](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201230162917205.png)

![image-20201230162951930](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201230162951930.png)

![image-20201230163021838](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201230163021838.png)

![image-20201230163052311](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201230163052311.png)

### 寄存器

![image-20201230084410766](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201230084410766.png)

### 跳转

长跳转、短跳转。详情见书本3.2.4.3。

![image-20201229221758564](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201229221758564.png)

![image-20201229221825170](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201229221825170.png)

### 特权级

#### RPL

RPL是请求特权级，是选择子的第0~~1位。

指令是资源的请求者。

指令“请求”、“访问”其他资源的能力等级，就是请求特权级。

不单独设置一条指令的请求特权级，而是用代码段为单位来设置这个代码段内的所有指令的特权级。

指令特权级存储在该指令所在代码段的选择子的RPL中。

CPU的当前特权级，就是正在执行的指令的请求特权级，所以，cs.RPL就是CPU的当前特权级。

#### 全称

DPL：Descriptor Privilege Level

RPL：Request Privilege Level

CPL：Current Privilege Level

#### 权限检查规则

![image-20201229194450054](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201229194450054.png)

![image-20201229221154337](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201229221154337.png)

#### 一致性代码

![image-20201229195107246](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201229195107246.png)

#### 问题

1. CPU的当前特权级，是不是CPL？

2. 选择子中有RPL，段描述符中有DPL。一个段描述符的选择子，为何都需要特权等级属性？
   1. RPL和DPL不总是相等。
   2. RPL是真正请求资源的访问者的权限。在各种门中，指向目标代码段描述符的选择子。当前门描述符的RPL和目标代码段的DPL不是同一个代码段的。
   3. 通过门，能让低特权级调用高特权级。为啥可以？我不记得具体过程。想知道，可以看《操作系统真相还原》5.4.5。

### 一致与非一致

![image-20201229171249950](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201229171249950.png)

完全不懂这个图。

### TSS

#### TSS结构图

![image-20201229173858214](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201229173858214.png)

TSS，任务状态段。

SS0、SS1、SS2分别是特权级0、1、2的栈，SS是当前使用的栈。

1. 低特权级向高特权级转移，会挑选TSS中的目标代码段的栈，即从SS0、SS1、SS2中选择一个，并将自身代码段的返回地址由CPU压入目标代码段的栈中。在调用函数的汇编代码中，我见过了。
2. 高特权级向低特权级返回，用 ret 、reft 等命令完成，从自己使用的栈中取出低特权级的代码段的地址，返回。由CPU自动完成。在返回前，需要将当前使用的栈更新为低特权级代码的栈。

#### TR

TR是寄存器，类似GDT的寄存器gdtr。

### 分页

#### 分页机制示意图

![image-20201230064426687](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201230064426687.png)

图画得很清楚，但图上面的文字写得很不好懂。我到现在还没理解它的意思，觉得它是错误的。《操作系统真相还原》中对分页机制的讲解就非常好懂。

#### 页目录项和页表项结构

![image-20201230071243833](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201230071243833.png)

#### 页目录表和页表的关系、内存布局

![image-20201230073301439](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201230073301439.png)

#### TLB

![image-20201230073824810](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201230073824810.png)

#### 启用分页机制

只需做好三件事：

![image-20201230072003022](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201230072003022.png)

##### cr3

![image-20201230071623444](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201230071623444.png)

##### cr0

![image-20201230072419504](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201230072419504.png)

![image-20201230072510197](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201230072510197.png)

## 代码理解

```assembly
; 初始化 32 位代码段描述符
	xor	eax, eax
	mov	ax, cs
	shl	eax, 4
	add	eax, LABEL_SEG_CODE32		; 段基址
	; LABEL_DESC_CODE32 + 2 内存地址从低到高移动16位，给16--31位赋值。ax是低16位的数据，并非mov ax,cs中的cs值。
	mov	word [LABEL_DESC_CODE32 + 2], ax
	; shr eax, 16 是高16位数据。在这里卡了很久，觉得除以2的16次方是咋回事。不必考虑这么多。总之，要把段基址拆成3部分：
	; 0~15、16~23、24~31填充到gdt中的段描述符中，不能重复。在上一句，已经通过ax设置了0~15，剩余的，需要用高16位设置。
	shr	eax, 16
	; 内存地址的移动单位是字节，故 LABEL_DESC_CODE32 + 2 是移动16位。
	; LABEL_DESC_CODE32 + 4 内存地址从低到高移动8 * 4 = 32位。给32--39（高0--7位）赋值
	mov	byte [LABEL_DESC_CODE32 + 4], al
	; LABEL_DESC_CODE32 + 7 内存地址从低到高移动8 * 7 = 56位。给56--63（高24--31位）赋值。
	mov	byte [LABEL_DESC_CODE32 + 7], ah
```

#### 段描述符格式图

![image-20201226230803303](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201226230803303.png)

结合图理解代码。

#### 选择子结构图

![image-20201227000139645](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201227000139645.png)



![image-20201227000756375](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201227000756375.png)

选择子是0x8，是GDT中的第几个段描述符呢？

0x8的二进制形式是`0x0000 0000 0000 1000`，第3位T1，值为0，表示是在GDT而不是LDT中的索引段描述符，高13位的值是1（十进制），故在GDT中的索引是1。

```assembly
[SECTION .gdt]
; GDT
;                              段基址,      段界限     , 属性
LABEL_GDT:	   Descriptor       0,                0, 0           ; 空描述符
LABEL_DESC_CODE32: Descriptor       0, SegCode32Len - 1, DA_C + DA_32; 非一致代码段
LABEL_DESC_VIDEO:  Descriptor 0B8000h,           0ffffh, DA_DRW	     ; 显存首地址
; GDT 结束

GdtLen		equ	$ - LABEL_GDT	; GDT长度
; GdtPtr是双字，32位。若定义为单字，[GdtPtr+2]，越界？
GdtPtr		dw	GdtLen - 1	; GDT界限
; dd定义双字类型变量，一个双字数据占4个字节单元，读完一个，偏移量加4
dd	0		; GDT基地址

; GDT 选择子
; GDT 选择子是段描述符在GDT中的索引，居然是通过内存偏移地址计算出来的。书上的文字说明和代码应该互相补充。
SelectorCode32		equ	LABEL_DESC_CODE32	- LABEL_GDT
SelectorVideo		equ	LABEL_DESC_VIDEO	- LABEL_GDT
; END of [SECTION .gdt]

[SECTION .s16]
[BITS	16]
LABEL_BEGIN:
	mov	ax, cs
	mov	ds, ax
	mov	es, ax
	mov	ss, ax
	mov	sp, 0100h

	; 在代码中，根据选择子定位代码的方法：
	; 1. 根据选择子定义找出它对应的段描述符；
	; 2.在段描述符赋值代码中根据段描述符定位它关联的代码。例如，LABEL_DESC_CODE32--->LABEL_SEG_CODE32。
	; 段描述符的基址就是代码标签所指代的内存地址 LABEL_SEG_CODE32。
	; 初始化 32 位代码段描述符
	xor	eax, eax
	mov	ax, cs
	shl	eax, 4
	add	eax, LABEL_SEG_CODE32
	mov	word [LABEL_DESC_CODE32 + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_CODE32 + 4], al
	mov	byte [LABEL_DESC_CODE32 + 7], ah

	; 为加载 GDTR 作准备
	xor	eax, eax
	; ds 是 cs，将cs左移动四位，然后加上偏移量。偏移量是LABEL_GDT，理解起来有点费事。
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_GDT		; eax <- gdt 基地址
	; dword全称Double Word
	; 右移动2个字节，这是由GdtPtr的结构（高32位是段基址，低16位是GDT界限）决定的。
	; 16 + 32，dword构造了32位。
	; gdtr的界限，即低16位，没有设置，为0。我在这上面还纠结了一会。
	mov	dword [GdtPtr + 2], eax	; [GdtPtr + 2] <- gdt 基地址

	; 加载 GDTR
	lgdt	[GdtPtr]

	; 关中断
	cli

	; 打开地址线A20
	in	al, 92h
	or	al, 00000010b
	out	92h, al

	; 准备切换到保护模式
	mov	eax, cr0
	or	eax, 1
	mov	cr0, eax

	; 真正进入保护模式
	jmp	dword SelectorCode32:0	; 执行这一句会把 SelectorCode32 装入 cs,
					; 并跳转到 Code32Selector:0  处
; END of [SECTION .s16]


[SECTION .s32]; 32 位代码段. 由实模式跳入.
[BITS	32]

LABEL_SEG_CODE32:
	mov	ax, SelectorVideo
	mov	gs, ax			; 视频段选择子(目的)

	mov	edi, (80 * 11 + 79) * 2	; 屏幕第 11 行, 第 79 列。
	mov	ah, 0Ch			; 0000: 黑底    1100: 红字
	mov	al, 'p'
	mov	[gs:edi], ax

	; 到此停止
	jmp	$

SegCode32Len	equ	$ - LABEL_SEG_CODE32
; END of [SECTION .s32]
```

### chapter3

#### a

```assembly
; ==========================================
; pmtest1.asm
; 编译方法：nasm pmtest1.asm -o pmtest1.bin
; ==========================================

%include	"pm.inc"	; 常量, 宏, 以及一些说明

	org	0100h
	jmp	LABEL_BEGIN

[SECTION .gdt]
; GDT
;                              段基址,      段界限     , 属性
LABEL_GDT:	   Descriptor       0,                0, 0           ; 空描述符
LABEL_DESC_CODE32: Descriptor       0, SegCode32Len - 1, DA_C + DA_32; 非一致代码段
LABEL_DESC_VIDEO:  Descriptor 0B8000h,           0ffffh, DA_DRW	     ; 显存首地址
; GDT 结束

GdtLen		equ	$ - LABEL_GDT	; GDT长度
GdtPtr		dw	GdtLen - 1	; GDT界限
		dd	0		; GDT基地址

; GDT 选择子
SelectorCode32		equ	LABEL_DESC_CODE32	- LABEL_GDT
SelectorVideo		equ	LABEL_DESC_VIDEO	- LABEL_GDT
; END of [SECTION .gdt]

[SECTION .s16]
[BITS	16]
LABEL_BEGIN:
	mov	ax, cs
	mov	ds, ax
	mov	es, ax
	mov	ss, ax
	mov	sp, 0100h

	; 初始化 32 位代码段描述符
	xor	eax, eax
	mov	ax, cs
	shl	eax, 4
	add	eax, LABEL_SEG_CODE32
	mov	word [LABEL_DESC_CODE32 + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_CODE32 + 4], al
	mov	byte [LABEL_DESC_CODE32 + 7], ah

	; 为加载 GDTR 作准备
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_GDT		; eax <- gdt 基地址
	mov	dword [GdtPtr + 2], eax	; [GdtPtr + 2] <- gdt 基地址

	; 加载 GDTR
	lgdt	[GdtPtr]

	; 关中断
	cli

	; 打开地址线A20
	in	al, 92h
	or	al, 00000010b
	out	92h, al

	; 准备切换到保护模式
	mov	eax, cr0
	or	eax, 1
	mov	cr0, eax

	; 真正进入保护模式
	jmp	dword SelectorCode32:0	; 执行这一句会把 SelectorCode32 装入 cs,
					; 并跳转到 Code32Selector:0  处
; END of [SECTION .s16]


[SECTION .s32]; 32 位代码段. 由实模式跳入.
[BITS	32]

LABEL_SEG_CODE32:
	mov	ax, SelectorVideo
	mov	gs, ax			; 视频段选择子(目的)

	mov	edi, (80 * 11 + 79) * 2	; 屏幕第 11 行, 第 79 列。
	mov	ah, 0Ch			; 0000: 黑底    1100: 红字
	mov	al, 'p'
	mov	[gs:edi], ax

	; 到此停止
	jmp	$

SegCode32Len	equ	$ - LABEL_SEG_CODE32
; END of [SECTION .s32]
```

#### b

![image-20201228205716618](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201228205716618.png)

```assembly
; ==========================================
; pmtest2.asm
; 编译方法：nasm pmtest2.asm -o pmtest2.com
; ==========================================

%include	"pm.inc"	; 常量, 宏, 以及一些说明

	org	0100h
	jmp	LABEL_BEGIN

[SECTION .gdt]
; GDT
;                            段基址,        段界限 , 属性
LABEL_GDT:         Descriptor    0,              0, 0         ; 空描述符
LABEL_DESC_NORMAL: Descriptor    0,         0ffffh, DA_DRW    ; Normal 描述符
LABEL_DESC_CODE32: Descriptor    0, SegCode32Len-1, DA_C+DA_32; 非一致代码段, 32
LABEL_DESC_CODE16: Descriptor    0,         0ffffh, DA_C      ; 非一致代码段, 16
LABEL_DESC_DATA:   Descriptor    0,      DataLen-1, DA_DRW    ; Data
LABEL_DESC_STACK:  Descriptor    0,     TopOfStack, DA_DRWA+DA_32; Stack, 32 位
LABEL_DESC_TEST:   Descriptor 0500000h,     0ffffh, DA_DRW	;段基址是5M
LABEL_DESC_VIDEO:  Descriptor  0B8000h,     0ffffh, DA_DRW    ; 显存首地址
; GDT 结束

GdtLen		equ	$ - LABEL_GDT	; GDT长度
GdtPtr		dw	GdtLen - 1	; GDT界限
		dd	0		; GDT基地址

; GDT 选择子
SelectorNormal		equ	LABEL_DESC_NORMAL	- LABEL_GDT
SelectorCode32		equ	LABEL_DESC_CODE32	- LABEL_GDT
SelectorCode16		equ	LABEL_DESC_CODE16	- LABEL_GDT
SelectorData		equ	LABEL_DESC_DATA		- LABEL_GDT
SelectorStack		equ	LABEL_DESC_STACK	- LABEL_GDT
SelectorTest		equ	LABEL_DESC_TEST		- LABEL_GDT
SelectorVideo		equ	LABEL_DESC_VIDEO	- LABEL_GDT
; END of [SECTION .gdt]

[SECTION .data1]	 ; 数据段
ALIGN	32
[BITS	32]
LABEL_DATA:
SPValueInRealMode	dw	0
; 字符串
PMMessage:		db	"In Protect Mode now. ^-^", 0	; 在保护模式中显示
OffsetPMMessage		equ	PMMessage - $$
StrTest:		db	"ABCDEFGHIJKLMNOPQRSTUVWXYZ", 0
OffsetStrTest		equ	StrTest - $$
DataLen			equ	$ - LABEL_DATA
; END of [SECTION .data1]


; 全局堆栈段
[SECTION .gs]
ALIGN	32
[BITS	32]
LABEL_STACK:
	times 512 db 0

TopOfStack	equ	$ - LABEL_STACK - 1

; END of [SECTION .gs]


[SECTION .s16]
[BITS	16]
LABEL_BEGIN:
	mov	ax, cs
	mov	ds, ax
	mov	es, ax
	mov	ss, ax
	mov	sp, 0100h

	;LABEL_GO_BACK_TO_REAL在330行定义了一条跳转机器指令。本语句直接更改那条机器指令的Segment，即段基址部分。
	;执行那条机器指令时，段基址即cs是ax，即本SECTION的cs。
	mov	[LABEL_GO_BACK_TO_REAL+3], ax
	mov	[SPValueInRealMode], sp

	; 初始化 16 位代码段描述符
	mov	ax, cs
	movzx	eax, ax
	shl	eax, 4
	add	eax, LABEL_SEG_CODE16
	mov	word [LABEL_DESC_CODE16 + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_CODE16 + 4], al
	mov	byte [LABEL_DESC_CODE16 + 7], ah

	; 初始化 32 位代码段描述符
	xor	eax, eax
	mov	ax, cs
	shl	eax, 4
	add	eax, LABEL_SEG_CODE32
	mov	word [LABEL_DESC_CODE32 + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_CODE32 + 4], al
	mov	byte [LABEL_DESC_CODE32 + 7], ah

	; 初始化数据段描述符
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_DATA
	mov	word [LABEL_DESC_DATA + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_DATA + 4], al
	mov	byte [LABEL_DESC_DATA + 7], ah

	; 初始化堆栈段描述符
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_STACK
	mov	word [LABEL_DESC_STACK + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_STACK + 4], al
	mov	byte [LABEL_DESC_STACK + 7], ah

	; 为加载 GDTR 作准备
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_GDT		; eax <- gdt 基地址
	; 把GDT的物理地址作为gdtr的段基地址
	mov	dword [GdtPtr + 2], eax	; [GdtPtr + 2] <- gdt 基地址

	; 加载 GDTR
	lgdt	[GdtPtr]

	; 关中断
	cli

	; 打开地址线A20
	in	al, 92h
	or	al, 00000010b
	out	92h, al

	; 准备切换到保护模式
	mov	eax, cr0
	or	eax, 1
	mov	cr0, eax

	; 真正进入保护模式
	jmp	dword SelectorCode32:0	; 执行这一句会把 SelectorCode32 装入 cs, 并跳转到 Code32Selector:0  处

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

LABEL_REAL_ENTRY:		; 从保护模式跳回到实模式就到了这里
	mov	ax, cs
	mov	ds, ax
	mov	es, ax
	mov	ss, ax

	mov	sp, [SPValueInRealMode]

	in	al, 92h		; `.
	and	al, 11111101b	;  | 关闭 A20 地址线
	out	92h, al		; /

	sti			; 开中断

	mov	ax, 4c00h	; `.
	int	21h		; /  回到 DOS
; END of [SECTION .s16]


[SECTION .s32]; 32 位代码段. 由实模式跳入.
[BITS	32]

LABEL_SEG_CODE32:
	mov	ax, SelectorData
	mov	ds, ax			; 数据段选择子
	mov	ax, SelectorTest
	mov	es, ax			; 测试段选择子
	mov	ax, SelectorVideo
	mov	gs, ax			; 视频段选择子

	mov	ax, SelectorStack
	mov	ss, ax			; 堆栈段选择子

	mov	esp, TopOfStack


	; 下面显示一个字符串
	mov	ah, 0Ch			; 0000: 黑底    1100: 红字
	xor	esi, esi		; esi清零。假如某位是1，结果是0；若某位是0，结果是0。所以，结果必然是每位都是0。
	xor	edi, edi
	mov	esi, OffsetPMMessage	; 源数据偏移
	mov	edi, (80 * 10 + 0) * 2	; 目的数据偏移。屏幕第 10 行, 第 0 列。
	cld;告诉字符串向前移动
.1:
	lodsb ;将字符串中的si指针所指向的一个字节装入al中，并增加或减小edi。
	test	al, al;判断al是否为空  为空跳转到2
	jz	.2
	mov	[gs:edi], ax;gs是视频，不为空显示当前字符
	add	edi, 2
	jmp	.1
.2:	; 显示完毕

	call	DispReturn;模拟换行

	call	TestRead
	call	TestWrite
	call	TestRead

	; 到此停止
	jmp	SelectorCode16:0

; ------------------------------------------------------------------------
TestRead:
	xor	esi, esi
	mov	ecx, 8
.loop:
	mov	al, [es:esi];   从新增的5MB开始地址处开始
	call	DispAL
	inc	esi
	loop	.loop

	call	DispReturn

	ret
; TestRead 结束-----------------------------------------------------------


; ------------------------------------------------------------------------
TestWrite:
	push	esi
	push	edi
	xor	esi, esi
	xor	edi, edi
	mov	esi, OffsetStrTest	; 源数据偏移
	cld
.1:
	lodsb
	test	al, al		; 检测是否为空
	jz	.2					; 如果al不是0，跳转到2。
	mov	[es:edi], al
	inc	edi
	jmp	.1
.2:

	pop	edi
	pop	esi

	ret
; TestWrite 结束----------------------------------------------------------


; ------------------------------------------------------------------------
; 显示 AL 中的数字
; 默认地:
;	数字已经存在 AL 中
;	edi 始终指向要显示的下一个字符的位置
; 被改变的寄存器:
;	ax, edi
; ------------------------------------------------------------------------
DispAL:
	push	ecx
	push	edx

	mov	ah, 0Ch			; 0000: 黑底    1100: 红字
	mov	dl, al			; 把al存储到dl中，在下面会改变al，当操作结束时，再从dl中取出旧al值。
	shr	al, 4				; 将al右移动四位，现在al只保留了高四位。
	mov	ecx, 2			; 循环两次。字母ABCD的十六进制只需要两个字符表示，如：A-----41H
.begin:
	and	al, 01111b	; 只保留四位。例如，0x41，划分为4和1两部分进行处理。
	cmp	al, 9				; al 比 9大，跳转到1;否则，不跳转。
	ja	.1
	add	al, '0'			; 4 + ‘0’,结果是字符串 '4' 。
	jmp	.2
.1:
	sub	al, 0Ah		  ; 比十进制数10大多少，然后加上A，这是把al的值转成字符的形式，比如'B'。
	add	al, 'A'
.2:
	mov	[gs:edi], ax	; 显示数据
	add	edi, 2				; 间距为何是2？

	mov	al, dl				; 再次从dl即al的低四位中获取41H的第二部分。
	loop	.begin
	add	edi, 2

	pop	edx
	pop	ecx

	ret
; DispAL 结束-------------------------------------------------------------

;80*25彩色字模式的显示显存在内存中的地址为B8000h~BFFFH,共32k.
;向这个地址写入的内容立即显示在屏幕上边.在80*25彩色字模式 下共可以显示25行,
;每行80字符,每个字符在显存中占两个字节,
;第一个字节是字符的ASCII码.第二字节是字符的属性，(80字符占160个字节）。



; ------------------------------------------------------------------------
DispReturn:			;换行符，理解不了
	push	eax
	push	ebx
	mov	eax, edi
	mov	bl, 160
	div	bl;很复杂的div指令，搞不懂.; eax/bl 执行后al＝当前行号
	and	eax, 0FFh; 只保留行号，列号清0
	inc	eax ; eax+＝1，使eax为当前行的下一行
	mov	bl, 160
	mul	bl ; eax*bl，eax为当前行的下一行的开始
	mov	edi, eax; 使edi指向当前行的下一行的开始
	pop	ebx
	pop	eax

	ret
; DispReturn 结束---------------------------------------------------------

SegCode32Len	equ	$ - LABEL_SEG_CODE32
; END of [SECTION .s32]


; 16 位代码段. 由 32 位代码段跳入, 跳出后到实模式
[SECTION .s16code]
ALIGN	32
[BITS	16]
LABEL_SEG_CODE16:
	; 跳回实模式:
	mov	ax, SelectorNormal
	mov	ds, ax
	mov	es, ax
	mov	fs, ax
	mov	gs, ax
	mov	ss, ax

	mov	eax, cr0
	and	al, 11111110b
	mov	cr0, eax

LABEL_GO_BACK_TO_REAL:
	jmp	0:LABEL_REAL_ENTRY	; 段地址会在程序开始处被设置成正确的值

Code16Len	equ	$ - LABEL_SEG_CODE16

; END of [SECTION .s16code]
```



```assembly
mov	ax, cs
mov	ds, ax
mov	es, ax
mov	ss, ax
mov	sp, 0100h

mov	[LABEL_GO_BACK_TO_REAL+3], ax;LABEL_GO_BACK_TO_REAL在330行定义了，在这里就可以直接用吗？
mov	[SPValueInRealMode], sp

; some code


; 16 位代码段. 由 32 位代码段跳入, 跳出后到实模式
[SECTION .s16code]
ALIGN	32
[BITS	16]
LABEL_SEG_CODE16:
	; 跳回实模式:
	mov	ax, SelectorNormal
	mov	ds, ax
	mov	es, ax
	mov	fs, ax
	mov	gs, ax
	mov	ss, ax

	mov	eax, cr0
	and	al, 11111110b
	mov	cr0, eax

LABEL_GO_BACK_TO_REAL:
	jmp	0:LABEL_REAL_ENTRY	; 段地址会在程序开始处被设置成正确的值

Code16Len	equ	$ - LABEL_SEG_CODE16

; END of [SECTION .s16code]
```

![image-20201228204155231](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201228204155231.png)

根据这张图，LABEL_GO_BACK_TO_REAL 指代一条机器指令，`mov	[LABEL_GO_BACK_TO_REAL+3], ax;` 更改了这条机器码的的Segment部分，即段基址，是cs。这条指令的构成是：操作码 + 偏移量 + 段基址。

#### c

```assembly
; ==========================================
; pmtest3.asm
; 编译方法：nasm pmtest3.asm -o pmtest3.com
; ==========================================

%include	"pm.inc"	; 常量, 宏, 以及一些说明

org	0100h
	jmp	LABEL_BEGIN

[SECTION .gdt]
; GDT
;                                         段基址,       段界限     , 属性
LABEL_GDT:         Descriptor       0,                 0, 0     	; 空描述符
LABEL_DESC_NORMAL: Descriptor       0,            0ffffh, DA_DRW	; Normal 描述符
LABEL_DESC_CODE32: Descriptor       0,  SegCode32Len - 1, DA_C + DA_32	; 非一致代码段, 32
LABEL_DESC_CODE16: Descriptor       0,            0ffffh, DA_C		; 非一致代码段, 16
LABEL_DESC_DATA:   Descriptor       0,       DataLen - 1, DA_DRW+DA_DPL1	; Data
LABEL_DESC_STACK:  Descriptor       0,        TopOfStack, DA_DRWA + DA_32; Stack, 32 位
LABEL_DESC_LDT:    Descriptor       0,        LDTLen - 1, DA_LDT	; LDT
LABEL_DESC_VIDEO:  Descriptor 0B8000h,            0ffffh, DA_DRW	; 显存首地址
; GDT 结束

GdtLen		equ	$ - LABEL_GDT	; GDT长度
GdtPtr		dw	GdtLen - 1	; GDT界限
		dd	0		; GDT基地址

; GDT 选择子
SelectorNormal		equ	LABEL_DESC_NORMAL	- LABEL_GDT
SelectorCode32		equ	LABEL_DESC_CODE32	- LABEL_GDT
SelectorCode16		equ	LABEL_DESC_CODE16	- LABEL_GDT
SelectorData		equ	LABEL_DESC_DATA		- LABEL_GDT
SelectorStack		equ	LABEL_DESC_STACK	- LABEL_GDT
SelectorLDT		equ	LABEL_DESC_LDT		- LABEL_GDT
SelectorVideo		equ	LABEL_DESC_VIDEO	- LABEL_GDT
; END of [SECTION .gdt]

[SECTION .data1]	 ; 数据段
ALIGN	32
[BITS	32]
LABEL_DATA:
SPValueInRealMode	dw	0
; 字符串
PMMessage:		db	"In Protect Mode now. ^-^", 0	; 进入保护模式后显示此字符串
OffsetPMMessage		equ	PMMessage - $$
StrTest:		db	"ABCDEFGHIJKLMNOPQRSTUVWXYZ", 0
OffsetStrTest		equ	StrTest - $$
DataLen			equ	$ - LABEL_DATA
; END of [SECTION .data1]


; 全局堆栈段
[SECTION .gs]
ALIGN	32
[BITS	32]
LABEL_STACK:
	times 512 db 0

TopOfStack	equ	$ - LABEL_STACK - 1

; END of [SECTION .gs]


[SECTION .s16]
[BITS	16]
LABEL_BEGIN:
	mov	ax, cs
	mov	ds, ax
	mov	es, ax
	mov	ss, ax
	mov	sp, 0100h

	mov	[LABEL_GO_BACK_TO_REAL+3], ax
	mov	[SPValueInRealMode], sp

	; 初始化 16 位代码段描述符
	mov	ax, cs
	movzx	eax, ax
	shl	eax, 4
	add	eax, LABEL_SEG_CODE16
	mov	word [LABEL_DESC_CODE16 + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_CODE16 + 4], al
	mov	byte [LABEL_DESC_CODE16 + 7], ah

	; 初始化 32 位代码段描述符
	xor	eax, eax
	mov	ax, cs
	shl	eax, 4
	add	eax, LABEL_SEG_CODE32
	mov	word [LABEL_DESC_CODE32 + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_CODE32 + 4], al
	mov	byte [LABEL_DESC_CODE32 + 7], ah

	; 初始化数据段描述符
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_DATA
	mov	word [LABEL_DESC_DATA + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_DATA + 4], al
	mov	byte [LABEL_DESC_DATA + 7], ah

	; 初始化堆栈段描述符
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_STACK
	mov	word [LABEL_DESC_STACK + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_STACK + 4], al
	mov	byte [LABEL_DESC_STACK + 7], ah

	; 初始化 LDT 在 GDT 中的描述符
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_LDT
	mov	word [LABEL_DESC_LDT + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_LDT + 4], al
	mov	byte [LABEL_DESC_LDT + 7], ah

	; 初始化 LDT 中的描述符
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_CODE_A
	mov	word [LABEL_LDT_DESC_CODEA + 2], ax
	shr	eax, 16
	mov	byte [LABEL_LDT_DESC_CODEA + 4], al
	mov	byte [LABEL_LDT_DESC_CODEA + 7], ah

	; 为加载 GDTR 作准备
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_GDT		; eax <- gdt 基地址
	mov	dword [GdtPtr + 2], eax	; [GdtPtr + 2] <- gdt 基地址

	; 加载 GDTR
	lgdt	[GdtPtr]

	; 关中断
	cli

	; 打开地址线A20
	in	al, 92h
	or	al, 00000010b
	out	92h, al

	; 准备切换到保护模式
	mov	eax, cr0
	or	eax, 1
	mov	cr0, eax

	; 真正进入保护模式
	jmp	dword SelectorCode32:0	; 执行这一句会把 SelectorCode32 装入 cs, 并跳转到 Code32Selector:0  处

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

LABEL_REAL_ENTRY:		; 从保护模式跳回到实模式就到了这里
	mov	ax, cs
	mov	ds, ax
	mov	es, ax
	mov	ss, ax

	mov	sp, [SPValueInRealMode]

	in	al, 92h		; ┓
	and	al, 11111101b	; ┣ 关闭 A20 地址线
	out	92h, al		; ┛

	sti			; 开中断

	mov	ax, 4c00h	; ┓
	int	21h		; ┛回到 DOS
; END of [SECTION .s16]


[SECTION .s32]; 32 位代码段. 由实模式跳入.
[BITS	32]

LABEL_SEG_CODE32:
	mov	ax, SelectorData
	mov	ds, ax			; 数据段选择子
	mov	ax, SelectorVideo
	mov	gs, ax			; 视频段选择子

	mov	ax, SelectorStack
	mov	ss, ax			; 堆栈段选择子

	mov	esp, TopOfStack


	; 下面显示一个字符串
	mov	ah, 0Ch			; 0000: 黑底    1100: 红字
	xor	esi, esi
	xor	edi, edi
	mov	esi, OffsetPMMessage	; 源数据偏移
	mov	edi, (80 * 10 + 0) * 2	; 目的数据偏移。屏幕第 10 行, 第 0 列。
	cld
.1:
	lodsb
	test	al, al
	jz	.2
	mov	[gs:edi], ax
	add	edi, 2
	jmp	.1
.2:	; 显示完毕

	call	DispReturn

	; Load LDT
	mov	ax, SelectorLDT
	lldt	ax

	jmp	SelectorLDTCodeA:0	; 跳入局部任务

; ------------------------------------------------------------------------
DispReturn:
	push	eax
	push	ebx
	mov	eax, edi
	mov	bl, 160
	div	bl
	and	eax, 0FFh
	inc	eax
	mov	bl, 160
	mul	bl
	mov	edi, eax
	pop	ebx
	pop	eax

	ret
; DispReturn 结束---------------------------------------------------------

SegCode32Len	equ	$ - LABEL_SEG_CODE32
; END of [SECTION .s32]


; 16 位代码段. 由 32 位代码段跳入, 跳出后到实模式
[SECTION .s16code]
ALIGN	32
[BITS	16]
LABEL_SEG_CODE16:
	; 跳回实模式:
	mov	ax, SelectorNormal
	mov	ds, ax
	mov	es, ax
	mov	fs, ax
	mov	gs, ax
	mov	ss, ax

	mov	eax, cr0
	and	al, 11111110b
	mov	cr0, eax

LABEL_GO_BACK_TO_REAL:
	jmp	0:LABEL_REAL_ENTRY	; 段地址会在程序开始处被设置成正确的值

Code16Len	equ	$ - LABEL_SEG_CODE16

; END of [SECTION .s16code]


; LDT
[SECTION .ldt]
ALIGN	32
LABEL_LDT:			; 不是LDT描述符，而是label。它的内存地址中至少包含32个0。
;                            段基址       段界限      属性
LABEL_LDT_DESC_CODEA: Descriptor 0, CodeALen - 1, DA_C + DA_32 ; Code, 32 位。它的内存地址中至少包含32个0。

LDTLen		equ	$ - LABEL_LDT

; SelectorLDTCodeA 的内存地址中至少包含32个0。
; equ 是对以某个内存地址开始的那段内存填充数据。
; align影响内存地址，但不影响内存中的数据。
; 假如一段内存的初始地址是4,这段内存中的数据占据32个字节或其他内存单位，那么，下一个变量被分配的内存地址，不是36，而是64。
; 内存地址，和内存中的页，并不矛盾。
; LDT 选择子
SelectorLDTCodeA	equ	LABEL_LDT_DESC_CODEA	- LABEL_LDT + SA_TIL
; END of [SECTION .ldt]


; CodeA (LDT, 32 位代码段)
[SECTION .la]
ALIGN	32
[BITS	32]
LABEL_CODE_A:
	mov	ax, SelectorVideo
	mov	gs, ax			; 视频段选择子(目的)

	mov	edi, (80 * 12 + 0) * 2	; 屏幕第 10 行, 第 0 列。
	mov	ah, 0Ch			; 0000: 黑底    1100: 红字
	mov	al, 'L'
	mov	[gs:edi], ax

	; 准备经由16位代码段跳回实模式
	jmp	SelectorCode16:0
CodeALen	equ	$ - LABEL_CODE_A
; END of [SECTION .la]
```



```assembly
; LDT 选择子
SelectorLDTCodeA	equ	LABEL_LDT_DESC_CODEA	- LABEL_LDT + SA_TIL
```

选择子是描述符索引，索引是描述符内存地址的偏移量。

若对align的理解是正确的，`LABEL_LDT_DESC_CODEA	- LABEL_LDT` 的值是一个内存地址，这个地址的低5位是0，加4，就是将低3位中的第3位即 `T1` 设置为 1。LDT和GDT的描述符的选择子结构相同，T1位是分辨二者的标识符。当T1是1时，CPU知道当前选择子是LDT中的描述符的选择子，会从LDT中获取段信息。

![image-20201229144912678](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201229144912678.png)

```assembly
; RPL(Requested Privilege Level): 请求特权级，用于特权检查。
;
; TI(Table Indicator): 引用描述符表指示位
;	TI=0 指示从全局描述符表GDT中读取描述符；
;	TI=1 指示从局部描述符表LDT中读取描述符。
;

;----------------------------------------------------------------------------
; 选择子类型值说明
; 其中:
;       SA_  : Selector Attribute

SA_RPL0		EQU	0	; ┓
SA_RPL1		EQU	1	; ┣ RPL
SA_RPL2		EQU	2	; ┃
SA_RPL3		EQU	3	; ┛

SA_TIG		EQU	0	; ┓TI
SA_TIL		EQU	4	; ┛
;----------------------------------------------------------------------------
```

#### f

````assembly
; 启动分页机制 --------------------------------------------------------------
SetupPaging:
	; 为简化处理, 所有线性地址对应相等的物理地址.

	; 首先初始化页目录
	mov	ax, SelectorPageDir	; 此段首地址为 PageDirBase
	mov	es, ax		; 不知道作用是什么。
	mov	ecx, 1024		; 共 1K 个表项。设置循环次数是1024次
	xor	edi, edi		; 设置为0
	xor	eax, eax
	mov	eax, PageTblBase | PG_P  | PG_USU | PG_RWW	; 设置页表结构的属性部分
.1:					; 用数字作标签并不好，完全可以改成.createPageDir
	stosd			; 将eax的值复制到edi指向的内存，并递增edi。即，将页表存储到一个字节中。
	; 为了简化, 所有页表在内存中是连续的。4096是2的12次方。每个页表大小是4KB，A、B页表相邻，B的页表地址是A的地址 + 4KB。
	; 页表结构的低12位是属性部分，加4096不破坏前面设置的页表结构的属性，只增加了页表的地址。
	add	eax, 4096
	loop	.1

	; 再初始化所有页表 (1K 个, 4M 内存空间)
	mov	ax, SelectorPageTbl	; 此段首地址为 PageTblBase
	mov	es, ax
	mov	ecx, 1024 * 1024	; 共 1M 个页表项, 也即有 1M 个页
	xor	edi, edi
	xor	eax, eax
	mov	eax, PG_P  | PG_USU | PG_RWW
.2:
	stosd
	add	eax, 4096		; 每一页指向 4K 的空间
	loop	.2

	mov	eax, PageDirBase
	mov	cr3, eax		; 设置cr3的值。cr3是crx系列寄存器（控制寄存器）之一，存储页目录表的物理地址
	mov	eax, cr0
	or	eax, 80000000h
	mov	cr0, eax
	jmp	short .3
.3:
	nop

	ret
; 分页机制启动完毕 ----------------------------------------------------------
````

### chapter4

#### a

```assembly

;%define	_BOOT_DEBUG_	; 做 Boot Sector 时一定将此行注释掉!将此行打开后用 nasm Boot.asm -o Boot.com 做成一个.COM文件易于调试

%ifdef	_BOOT_DEBUG_
	org  0100h			; 调试状态, 做成 .COM 文件, 可调试
%else
	org  07c00h			; Boot 状态, Bios 将把 Boot Sector 加载到 0:7C00 处并开始执行
%endif

	jmp short LABEL_START		; Start to boot.
	nop				; 这个 nop 不可少

	; 值能任意确定还是必须按规定填写固定的值？
	; 值中有十进制、十六进制
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

LABEL_START:
	mov	ax, cs
	mov	ds, ax
	mov	es, ax
	Call	DispStr			; 调用显示字符串例程
	jmp	$			; 无限循环
DispStr:
	mov	ax, BootMessage
	mov	bp, ax			; ES:BP = 串地址
	mov	cx, 16			; CX = 串长度
	；先指定 AH 寄存器
	; 01H：设置光标形状。40×25 16 色 文本。仍不明白这项的具体作用。???
	; 13H：在Teletype模式下显示字符串。640×480 256色
	; mov	ax, 01302h 会打印非常奇怪的颜色，看不清字符串。
	mov	ax, 01301h		; AH = 13,  AL = 01h
	; 页号有啥作用？百度不到。
	; BX显示模式属性。
	; mov     bx, 002ch 绿色背景
	; mov			bx,	032ch 一片黑色
	; mov			bx, 0c2ch	一片黑色
	mov	bx, 000ch		; 页号为0(BH = 0) 黑底红字(BL = 0Ch,高亮)
	; dl 为0，有啥作用？
	; DH＝行(Y坐标)
	; DL＝列 (X坐标)
	mov	dl, 0
	int	10h			; int 10h
	ret
BootMessage:		db	"Hello, OS world!"
times 	510-($-$$)	db	0	; 填充剩下的空间，使生成的二进制代码恰好为512字节
dw 	0xaa55				; 结束标志
```

```assembly
mov dh, 10
mov	dl, 10
int	10h			; int 10h
```

效果：

![image-20210102201951820](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20210102201951820.png)

```assembly
; BH，页号。不知道啥干啥的。
; BL，设置背景色和前景色。0d，背景色--黑，前景色--紫红
mov	bx, 000dh
```

![image-20210102210554287](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20210102210554287.png)

时间消耗点，int 10的调用。资料中说得不够清晰，密密麻麻一大页。

耗费时间1个小时20分。

1>创建新软盘pm.img，需要格式化，需要将pm.img设置为cg用户具有可写权限。

2>int 0x10调用。资料很不好，密密麻麻，太多了。调用模式是：

1. AH指定为13，表示打印字符串。
2. 让bp指向字符串。es:bp = 串地址。
3. DX，其实行列坐标。
4. BH，页号。
5. BL，被打印字符串的前景色与背景色。0d，是黑底紫红字；0C，是黑底红字。

资料地址：https://www.cnblogs.com/magic-cube/archive/2011/10/19/2217676.html

使用 INT 10H 中断服务程序时，先指定 AH 寄存器为下表编号其中之一，该编号表示欲调用的功用，而其他寄存器的详细说明，参考表后文字，当一切设定好之后再调用 INT 10H

![image-20210102211050335](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20210102211050335.png)

#### b

```assembly

;%define	_BOOT_DEBUG_	; 做 Boot Sector 时一定将此行注释掉!将此行打开后可用 nasm Boot.asm -o Boot.com 做成一个.COM文件易于调试

%ifdef	_BOOT_DEBUG_
	org  0100h			; 调试状态, 做成 .COM 文件, 可调试
%else
	org  07c00h			; Boot 状态, Bios 将把 Boot Sector 加载到 0:7C00 处并开始执行
%endif

;================================================================================================
%ifdef	_BOOT_DEBUG_
BaseOfStack		equ	0100h	; 调试状态下堆栈基地址(栈底, 从这个位置向低地址生长)
%else
BaseOfStack		equ	07c00h	; 堆栈基地址(栈底, 从这个位置向低地址生长)
%endif

BaseOfLoader		equ	09000h	; LOADER.BIN 被加载到的位置 ----  段地址
OffsetOfLoader		equ	0100h	; LOADER.BIN 被加载到的位置 ---- 偏移地址
RootDirSectors		equ	14	; 根目录占用空间
SectorNoOfRootDirectory	equ	19	; Root Directory 的第一个扇区号
;================================================================================================

	jmp short LABEL_START		; Start to boot.
	nop				; 这个 nop 不可少

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
	BPB_TotSec32	DD 0		; 如果 wTotalSectorCount 是 0 由这个值记录扇区数
	BS_DrvNum	DB 0		; 中断 13 的驱动器号
	BS_Reserved1	DB 0		; 未使用
	BS_BootSig	DB 29h		; 扩展引导标记 (29h)
	BS_VolID	DD 0		; 卷序列号
	BS_VolLab	DB 'OrangeS0.02'; 卷标, 必须 11 个字节
	BS_FileSysType	DB 'FAT12   '	; 文件系统类型, 必须 8个字节  

LABEL_START:
	mov	ax, cs
	mov	ds, ax			; 数据段
	mov	es, ax			; 附加段
	mov	ss, ax			; 栈段
	mov	sp, BaseOfStack	; 栈指针

	xor	ah, ah	; `.
	xor	dl, dl	;  |  软驱复位。模板。
	int	13h	; /
	
; 下面在 A 盘的根目录寻找 LOADER.BIN
;SectorNoOfRootDirectory	equ	19	; Root Directory 的第一个扇区号
	mov	word [wSectorNo], SectorNoOfRootDirectory
LABEL_SEARCH_IN_ROOT_DIR_BEGIN:
	; wRootDirSizeForLoop	dw	RootDirSectors	; Root Directory 占用的扇区数，在循环中会递减至零
	; RootDirSectors		equ	14	; 根目录占用空间。14这个值是怎么得来的?设置成15可以吗？
	cmp	word [wRootDirSizeForLoop], 0	;  `. 判断根目录区是不是已经读完
	jz	LABEL_NO_LOADERBIN		;  /  如果读完表示没有找到 LOADER.BIN。[wRootDirSizeForLoop]的值是0时，
	dec	word [wRootDirSizeForLoop]	; /
	mov	ax, BaseOfLoader
	mov	es, ax			; es <- BaseOfLoader
	mov	bx, OffsetOfLoader	; bx <- OffsetOfLoader
	mov	ax, [wSectorNo]		; ax <- Root Directory 中的某 Sector 号
	mov	cl, 1
	call	ReadSector

	mov	si, LoaderFileName	; ds:si -> "LOADER  BIN";si是啥？SI源变址寄存器。si的基址总是ds吗？
	; es:di -> BaseOfLoader:0100。di是啥？di 用于存放目的bai串偏移量，du被命名为“目的变址zhi寄存器”。
	; mov al,[bx+di+12] 在“基址变址寻址方式”中，bx 作为基址指针，di 作为变址指针；di的基址总是es吗？
	mov	di, OffsetOfLoader
	cld	; si递增
	mov	dx, 10h;	？为dx是10h？一个扇区512Kb，每个根目录条目32kb，512/32=10h。
LABEL_SEARCH_FOR_LOADERBIN:
	cmp	dx, 0				   ; `. 循环次数控制,dx == 0时，循环结束。
	jz	LABEL_GOTO_NEXT_SECTOR_IN_ROOT_DIR ;  / 如果已经读完了一个 Sector,dx == 0时，循环结束。
	dec	dx				   ; /  就跳到下一个 Sector
	mov	cx, 11			 ; LOADER  BIN 的长度是11。文件名8字节，扩展名3字节。
LABEL_CMP_FILENAME:
	cmp	cx, 0
	jz	LABEL_FILENAME_FOUND	; 如果比较了 11 个字符都相等, 表示找到
	dec	cx
	lodsb				; ds:si -> al
	cmp	al, byte [es:di]
	jz	LABEL_GO_ON
	jmp	LABEL_DIFFERENT		; 只要发现不一样的字符就表明本 DirectoryEntry
					; 不是我们要找的 LOADER.BIN
LABEL_GO_ON:
	inc	di
	jmp	LABEL_CMP_FILENAME	; 继续循环

LABEL_DIFFERENT:
	and	di, 0FFE0h		; else `. di &= E0 为了让它指向本条目开头。将低5位置0，因为一个根目录条目的大小是32字节。
	add	di, 20h			;       |	下一个根目录条目的开头。正好是DIR_Name。
	mov	si, LoaderFileName	;       | di += 20h  下一个目录条目
	jmp	LABEL_SEARCH_FOR_LOADERBIN;    /

LABEL_GOTO_NEXT_SECTOR_IN_ROOT_DIR:
	add	word [wSectorNo], 1	; 扇区号加1，在下一个扇区内查找
	jmp	LABEL_SEARCH_IN_ROOT_DIR_BEGIN

LABEL_NO_LOADERBIN:
	mov	dh, 2			; "No LOADER."
	call	DispStr			; 显示字符串
%ifdef	_BOOT_DEBUG_
	mov	ax, 4c00h		; `.
	int	21h			; /  没有找到 LOADER.BIN, 回到 DOS
%else
	jmp	$			; 没有找到 LOADER.BIN, 死循环在这里
%endif

LABEL_FILENAME_FOUND:			; 找到 LOADER.BIN 后便来到这里继续
	jmp	$			; 代码暂时停在这里



;============================================================================
;变量
wRootDirSizeForLoop	dw	RootDirSectors	; Root Directory 占用的扇区数，
						; 在循环中会递减至零
wSectorNo		dw	0		; 要读取的扇区号
bOdd			db	0		; 奇数还是偶数

;字符串
LoaderFileName		db	"LOADER  BIN", 0 ; LOADER.BIN 之文件名
; 为简化代码, 下面每个字符串的长度均为 MessageLength
MessageLength		equ	9
BootMessage:		db	"Booting  " ; 9字节, 不够则用空格补齐. 序号 0
Message1		db	"Ready.   " ; 9字节, 不够则用空格补齐. 序号 1
Message2		db	"No LOADER" ; 9字节, 不够则用空格补齐. 序号 2
;============================================================================


;----------------------------------------------------------------------------
; 函数名: DispStr
;----------------------------------------------------------------------------
; 作用:
;	显示一个字符串, 函数开始时 dh 中应该是字符串序号(0-based)
DispStr:
	mov	ax, MessageLength
	mul	dh
	add	ax, BootMessage
	mov	bp, ax			; `.
	mov	ax, ds			;  | ES:BP = 串地址
	mov	es, ax			; /
	mov	cx, MessageLength	; CX = 串长度
	mov	ax, 01301h		; AH = 13,  AL = 01h
	mov	bx, 0007h		; 页号为0(BH = 0) 黑底白字(BL = 07h)
	mov	dl, 0
	int	10h			; int 10h
	ret


;----------------------------------------------------------------------------
; 函数名: ReadSector
;----------------------------------------------------------------------------
; 作用:
;	从第 ax 个 Sector 开始, 将 cl 个 Sector 读入 es:bx 中。那为何能通过es:di来读取呢？
ReadSector:
	; -----------------------------------------------------------------------
	; 怎样由扇区号求扇区在磁盘中的位置 (扇区号 -> 柱面号, 起始扇区, 磁头号)
	; -----------------------------------------------------------------------
	; 设扇区号为 x
	;                           ┌ 柱面号 = y >> 1
	;       x           ┌ 商 y ┤
	; -------------- => ┤      └ 磁头号 = y & 1
	;  每磁道扇区数     │
	;                   └ 余 z => 起始扇区号 = z + 1
	push	bp
	mov	bp, sp
	sub	esp, 2 ; 辟出两个字节的堆栈区域保存要读的扇区数: byte [bp-2]

	mov	byte [bp-2], cl
	push	bx			; 保存 bx
	mov	bl, [BPB_SecPerTrk]	; bl: 除数
	div	bl			; y 在 al 中, z 在 ah 中
	inc	ah			; z ++
	mov	cl, ah			; cl <- 起始扇区号
	mov	dh, al			; dh <- y
	shr	al, 1			; y >> 1 (y/BPB_NumHeads)
	mov	ch, al			; ch <- 柱面号
	and	dh, 1			; dh & 1 = 磁头号
	pop	bx			; 恢复 bx
	; 至此, "柱面号, 起始扇区, 磁头号" 全部得到
	mov	dl, [BS_DrvNum]		; 驱动器号 (0 表示 A 盘)
.GoOnReading:
	mov	ah, 2			; 读
	mov	al, byte [bp-2]		; 读 al 个扇区
	int	13h
	jc	.GoOnReading		; 如果读取错误 CF 会被置为 1, 
					; 这时就不停地读, 直到正确为止
	add	esp, 2
	pop	bp

	ret

times 	510-($-$$)	db	0	; 填充剩下的空间，使生成的二进制代码恰好为512字节
dw 	0xaa55				; 结束标志
```

```assembly
;----------------------------------------------------------------------------
; 函数名: ReadSector
;----------------------------------------------------------------------------
; 作用:
;	从第 ax 个 Sector 开始, 将 cl 个 Sector 读入 es:bx 中
ReadSector:
	; -----------------------------------------------------------------------
	; 怎样由扇区号求扇区在磁盘中的位置 (扇区号 -> 柱面号, 起始扇区, 磁头号)
	; -----------------------------------------------------------------------
	; 设扇区号为 x
	;                           ┌ 柱面号 = y >> 1
	;       x           ┌ 商 y ┤
	; -------------- => ┤      └ 磁头号 = y & 1
	;  每磁道扇区数     │
	;                   └ 余 z => 起始扇区号 = z + 1
	push	bp
	mov	bp, sp
	sub	esp, 2 ; 辟出两个字节的堆栈区域保存要读的扇区数: byte [bp-2]

	mov	byte [bp-2], cl
	push	bx			; 保存 bx
	mov	bl, [BPB_SecPerTrk]	; bl: 除数
	div	bl			; y 在 al 中, z 在 ah 中
	inc	ah			; z ++
	mov	cl, ah			; cl <- 起始扇区号
	mov	dh, al			; dh <- y
	shr	al, 1			; y >> 1 (y/BPB_NumHeads)
	mov	ch, al			; ch <- 柱面号
	and	dh, 1			; dh & 1 = 磁头号
	pop	bx			; 恢复 bx
	; 至此, "柱面号, 起始扇区, 磁头号" 全部得到
	mov	dl, [BS_DrvNum]		; 驱动器号 (0 表示 A 盘)
.GoOnReading:
	mov	ah, 2			; 读
	mov	al, byte [bp-2]		; 读 al 个扇区
	int	13h
	; JC是判断C进位标志是否为1，为1则跳转到指定位置。
	jc	.GoOnReading		; 如果读取错误 CF 会被置为 1, 
					; 这时就不停地读, 直到正确为止
	add	esp, 2
	pop	bp

	ret
```

![image-20210102221938669](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20210102221938669.png)

又看了一次，几乎看懂了全部细节，可我仍感觉没有掌握。我能独自写出这些代码吗？不能。

#### c

遇到问题。不能加载boot.bin并运行。

![image-20210103084811556](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20210103084811556.png)

将a.img改成执行bochs的用户所有，并赋予读写执行权限，就解决了这个问题。

##### 数据区开始扇区号

数据区开始扇区号 = 根目录区开始扇区号 + 根目录区占用的扇区数 = 19 + 根目录区占用的扇区数

根目录占用的扇区数 = (根目录条目数 * 32) / 512 = (0xe0 * 32) / 512 = 14

根目录占用的扇区数根据商值向上取整，例如，实际结果是4.1、4.7，最终结果都是5。

耗费时间：1小12分。有一部分时间耗费在理解其他代码。原因是，没有从头到尾看书本讲解，和记忆力不好有关。计算公式使用占位符，而我看过占位符，却没能将占位符所表示的含义代入公式理解。看这些汇编代码，时刻弥漫着焦虑，代码难懂，又变得更难懂了。

##### GetFATEntry

```assembly
;----------------------------------------------------------------------------
; 函数名: GetFATEntry
;----------------------------------------------------------------------------
; 作用:
;	找到序号为 ax 的 Sector 在 FAT 中的条目, 结果放在 ax 中
;	需要注意的是, 中间需要读 FAT 的扇区到 es:bx 处, 所以函数一开始保存了 es 和 bx
GetFATEntry:
	push	es
	push	bx
	push	ax
	mov	ax, BaseOfLoader; `.
	sub	ax, 0100h	;  | 在 BaseOfLoader 后面留出 4K 空间用于存放 FAT
	mov	es, ax		; /
	pop	ax
	mov	byte [bOdd], 0
	mov	bx, 3
	mul	bx			; dx:ax = ax * 3
	mov	bx, 2
	div	bx			; dx:ax / 2  ==>  ax <- 商, dx <- 余数
	cmp	dx, 0
	jz	LABEL_EVEN
	mov	byte [bOdd], 1
LABEL_EVEN:;偶数
	; 现在 ax 中是 FATEntry 在 FAT 中的偏移量,下面来
	; 计算 FATEntry 在哪个扇区中(FAT占用不止一个扇区)
	xor	dx, dx			
	mov	bx, [BPB_BytsPerSec]
	div	bx ; dx:ax / BPB_BytsPerSec
		   ;  ax <- 商 (FATEntry 所在的扇区相对于 FAT 的扇区号)
		   ;  dx <- 余数 (FATEntry 在扇区内的偏移)
	push	dx
	mov	bx, 0 ; bx <- 0 于是, es:bx = (BaseOfLoader - 100):00
	add	ax, SectorNoOfFAT1 ; 此句之后的 ax 就是 FATEntry 所在的扇区号
	mov	cl, 2
	call	ReadSector ; 读取 FATEntry 所在的扇区, 一次读两个, 避免在边界
			   ; 发生错误, 因为一个 FATEntry 可能跨越两个扇区
	pop	dx
	add	bx, dx
	mov	ax, [es:bx]
	cmp	byte [bOdd], 1
	jnz	LABEL_EVEN_2
	shr	ax, 4
LABEL_EVEN_2:
	and	ax, 0FFFh

LABEL_GET_FAT_ENRY_OK:

	pop	bx
	pop	es
	ret
;----------------------------------------------------------------------------
```

FAT2是FAT1的备份。fat的第0项、第1项，不使用。

疑问：

1. fat的第2项表示数据区的第一个扇区。
2. 根目录条目中的DIR_FstClus的值是是文件开始簇号。一个簇只包含一个扇区，所以，根据簇号能计算出扇区号。

![image-20210103173037722](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20210103173037722.png)

划红线的句子，非常重要。它指出了获取文件存储在哪些扇区。

例如，FLOWER.TXT的DIR_FstClus的值是3，寻找该文件所占用的扇区应该从FAT表的第3个FAT项T开始寻找。T的值是0x008，值是8，下一个簇号是8。找到第8个FAT项，继续寻找下一个簇号。

簇号和FAT项的序号是对应的。这是最重要的一个规则。没啥理由。协议、规定而已。

FAT项中有两个特殊值：

1. 0xFF8。若FAT项的值大于或等于0XFF8，表示当前簇已经是本文件的最后一个簇。
2. 0xFF7。若FAT项的值等于0XFF7，表示当前簇是本文件的一个坏簇。
   1. 遇到坏簇怎么办？还能找到这个文件吗？

###### DeltaSectorNo

![image-20210103174058240](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20210103174058240.png)

我居然在这么明显的讲解这里疑惑了几分钟。

数据区的前两个簇，即前两个扇区不用，从引导扇区到数据区的第一个扇区（第0个、第1个、第2个中的第2个簇），相差14 + 19 + 2 - 2。

为什么需要减2？

为什么不是减4？

没想到好的理由。

数据区的第0个、第1个扇区，没有被使用。FAT表的前两个FAT项虽然也没被使用，但不是以扇区来计算的，无法减去。是否减去2，只是一个协议而已，只需要与FAT项的值对应即可。

耗费时间：1小时18分。实在想不出为啥理解这么慢，也许是因为，没能把所有内容自动连起来理解吧。

###### 代码理解

一个FAT项占用12个字节，一个扇区占用512个字节，512/12有余数，这意味着一个FAT项会跨域两个扇区。每次只读取一个扇区，然后从这个扇区里读取FAT项。

512 = 256 * 2 = 128 * 4 = 64 * 8 = 8 * 8 * 8 = 2的9次方 = 4 * （2的7次方）= 8 * 64 = 8 * （12 * 5 + 4）= 12 * 40 + 32 = 12 * 40 + 12 * 2 + 8 = 12 * 42 + 8。

一个扇区有42个FAT项 + 8个bit，最后一个FAT项需占用下一个扇区的4个bit。

1024 = 12 * 42 + 8 + 12 * 42 + 8 = 12 * 84 + 16 = 12 * 85 + 4。

二个扇区也不能保证最后一个FAT项在所读扇区内。

12 = 4 * 3 = 3 * 4 = 3 * （2的2次方）

```assembly

;%define	_BOOT_DEBUG_	; 做 Boot Sector 时一定将此行注释掉!将此行打开后用 nasm Boot.asm -o Boot.com 做成一个.COM文件易于调试

%ifdef	_BOOT_DEBUG_
	org  0100h			; 调试状态, 做成 .COM 文件, 可调试
%else
	org  07c00h			; Boot 状态, Bios 将把 Boot Sector 加载到 0:7C00 处并开始执行
%endif

;================================================================================================
%ifdef	_BOOT_DEBUG_
BaseOfStack		equ	0100h	; 调试状态下堆栈基地址(栈底, 从这个位置向低地址生长)
%else
BaseOfStack		equ	07c00h	; Boot状态下堆栈基地址(栈底, 从这个位置向低地址生长)
%endif

BaseOfLoader		equ	09000h	; LOADER.BIN 被加载到的位置 ----  段地址
OffsetOfLoader		equ	0100h	; LOADER.BIN 被加载到的位置 ---- 偏移地址

RootDirSectors		equ	14	; 根目录占用空间
SectorNoOfRootDirectory	equ	19	; Root Directory 的第一个扇区号
SectorNoOfFAT1		equ	1	; FAT1 的第一个扇区号 = BPB_RsvdSecCnt
DeltaSectorNo		equ	17	; DeltaSectorNo = BPB_RsvdSecCnt + (BPB_NumFATs * FATSz) - 2
					; 文件的开始Sector号 = DirEntry中的开始Sector号 + 根目录占用Sector数目 + DeltaSectorNo
;================================================================================================

	jmp short LABEL_START		; Start to boot.
	nop				; 这个 nop 不可少

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
	BPB_TotSec32	DD 0		; 如果 wTotalSectorCount 是 0 由这个值记录扇区数
	BS_DrvNum	DB 0		; 中断 13 的驱动器号
	BS_Reserved1	DB 0		; 未使用
	BS_BootSig	DB 29h		; 扩展引导标记 (29h)
	BS_VolID	DD 0		; 卷序列号
	BS_VolLab	DB 'OrangeS0.02'; 卷标, 必须 11 个字节
	BS_FileSysType	DB 'FAT12   '	; 文件系统类型, 必须 8个字节  

LABEL_START:	
	mov	ax, cs
	mov	ds, ax
	mov	es, ax
	mov	ss, ax
	mov	sp, BaseOfStack

	; 清屏
	mov	ax, 0600h		; AH = 6,  AL = 0h
	mov	bx, 0700h		; 黑底白字(BL = 07h)
	mov	cx, 0			; 左上角: (0, 0)
	mov	dx, 0184fh		; 右下角: (80, 50)
	int	10h			; int 10h

	mov	dh, 0			; "Booting  "
	call	DispStr			; 显示字符串
	
	xor	ah, ah	; ┓
	xor	dl, dl	; ┣ 软驱复位
	int	13h	; ┛
	
; 下面在 A 盘的根目录寻找 LOADER.BIN
	mov	word [wSectorNo], SectorNoOfRootDirectory
LABEL_SEARCH_IN_ROOT_DIR_BEGIN:
	cmp	word [wRootDirSizeForLoop], 0	; ┓
	jz	LABEL_NO_LOADERBIN		; ┣ 判断根目录区是不是已经读完
	dec	word [wRootDirSizeForLoop]	; ┛ 如果读完表示没有找到 LOADER.BIN
	mov	ax, BaseOfLoader
	mov	es, ax			; es <- BaseOfLoader
	mov	bx, OffsetOfLoader	; bx <- OffsetOfLoader	于是, es:bx = BaseOfLoader:OffsetOfLoader
	mov	ax, [wSectorNo]	; ax <- Root Directory 中的某 Sector 号
	mov	cl, 1
	call	ReadSector

	mov	si, LoaderFileName	; ds:si -> "LOADER  BIN"
	mov	di, OffsetOfLoader	; es:di -> BaseOfLoader:0100 = BaseOfLoader*10h+100
	cld
	mov	dx, 10h
LABEL_SEARCH_FOR_LOADERBIN:
	cmp	dx, 0										; ┓循环次数控制,
	jz	LABEL_GOTO_NEXT_SECTOR_IN_ROOT_DIR	; ┣如果已经读完了一个 Sector,
	dec	dx											; ┛就跳到下一个 Sector
	mov	cx, 11
LABEL_CMP_FILENAME:
	cmp	cx, 0
	jz	LABEL_FILENAME_FOUND	; 如果比较了 11 个字符都相等, 表示找到
dec	cx
	lodsb				; ds:si -> al
	cmp	al, byte [es:di]
	jz	LABEL_GO_ON
	jmp	LABEL_DIFFERENT		; 只要发现不一样的字符就表明本 DirectoryEntry 不是
; 我们要找的 LOADER.BIN
LABEL_GO_ON:
	inc	di
	jmp	LABEL_CMP_FILENAME	;	继续循环

LABEL_DIFFERENT:
	and	di, 0FFE0h						; else ┓	di &= E0 为了让它指向本条目开头
	add	di, 20h							;     ┃
	mov	si, LoaderFileName					;     ┣ di += 20h  下一个目录条目
	jmp	LABEL_SEARCH_FOR_LOADERBIN;    ┛

LABEL_GOTO_NEXT_SECTOR_IN_ROOT_DIR:
	add	word [wSectorNo], 1
	jmp	LABEL_SEARCH_IN_ROOT_DIR_BEGIN

LABEL_NO_LOADERBIN:
	mov	dh, 2			; "No LOADER."
	call	DispStr			; 显示字符串
%ifdef	_BOOT_DEBUG_
	mov	ax, 4c00h		; ┓
	int	21h			; ┛没有找到 LOADER.BIN, 回到 DOS
%else
	jmp	$			; 没有找到 LOADER.BIN, 死循环在这里
%endif

LABEL_FILENAME_FOUND:			; 找到 LOADER.BIN 后便来到这里继续
	mov	ax, RootDirSectors
	and	di, 0FFE0h		; di -> 当前条目的开始。为啥是0FFE0h？
	add	di, 01Ah		; di -> 首 Sector。DIR_FstClus相对当前条目的开始的偏移量是01Ah。
	mov	cx, word [es:di]	; 获取本文件的第一个簇号
	push	cx			; 保存此 Sector 在 FAT 中的序号
	; 根据簇号计算对应的扇区号 start
	add	cx, ax
	add	cx, DeltaSectorNo	; cl <- LOADER.BIN的起始扇区号(0-based)
	; 根据簇号计算对应的扇区号 end
	mov	ax, BaseOfLoader
	mov	es, ax			; es <- BaseOfLoader
	mov	bx, OffsetOfLoader	; bx <- OffsetOfLoader
	mov	ax, cx			; ax <- Sector 号

LABEL_GOON_LOADING_FILE:
	push	ax			; `. 此时ax是根据簇号所计算出来的扇区号。
	push	bx			;  |
	mov	ah, 0Eh			;  | 每读一个扇区就在 "Booting  " 后面
	mov	al, '.'			;  | 打一个点, 形成这样的效果:
	mov	bl, 0Fh			;  | Booting ......
	int	10h			;  |
	pop	bx			;  |
	pop	ax			; / 此时ax是根据簇号所计算出来的扇区号。供ReadSector使用。
	
	; 根据簇号计算出的扇区号把文件读入内存
	mov	cl, 1
	call	ReadSector
	pop	ax			; 取出此 Sector 在 FAT 中的序号，簇号，供GetFATEntry使用。
	; 读取该文件的下一个扇区号？还是簇号？
	; 调用此函数后，存储在ax中的是簇号，当前文件的剩余部分存储在该簇号内存中。
	call	GetFATEntry
	; ax == 0FFFh时，ZF == 1，jz跳转
	cmp	ax, 0FFFh
	jz	LABEL_FILE_LOADED
	; 保存 Sector 在 FAT 中的序号。在根据簇号计算该簇对应的扇区号前存储原始簇号。再次调用本函数时需要这个簇号当参数。
	; 比较奇特。当前参数，得出的结果是下一个参数。
	push	ax			
	mov	dx, RootDirSectors
	add	ax, dx
	add	ax, DeltaSectorNo
	add	bx, [BPB_BytsPerSec]
	jmp	LABEL_GOON_LOADING_FILE
LABEL_FILE_LOADED:

	mov	dh, 1			; "Ready."
	call	DispStr			; 显示字符串

; *****************************************************************************************************
	jmp	BaseOfLoader:OffsetOfLoader	; 这一句正式跳转到已加载到内
						; 存中的 LOADER.BIN 的开始处，
						; 开始执行 LOADER.BIN 的代码。
						; Boot Sector 的使命到此结束
; *****************************************************************************************************



;============================================================================
;变量
;----------------------------------------------------------------------------
wRootDirSizeForLoop	dw	RootDirSectors	; Root Directory 占用的扇区数, 在循环中会递减至零.
wSectorNo		dw	0		; 要读取的扇区号
bOdd			db	0		; 奇数还是偶数

;============================================================================
;字符串
;----------------------------------------------------------------------------
LoaderFileName		db	"LOADER  BIN", 0	; LOADER.BIN 之文件名
; 为简化代码, 下面每个字符串的长度均为 MessageLength
MessageLength		equ	9
BootMessage:		db	"Booting  "; 9字节, 不够则用空格补齐. 序号 0
Message1		db	"Ready.   "; 9字节, 不够则用空格补齐. 序号 1
Message2		db	"No LOADER"; 9字节, 不够则用空格补齐. 序号 2
;============================================================================


;----------------------------------------------------------------------------
; 函数名: DispStr
;----------------------------------------------------------------------------
; 作用:
;	显示一个字符串, 函数开始时 dh 中应该是字符串序号(0-based)
DispStr:
	mov	ax, MessageLength
	mul	dh
	add	ax, BootMessage
	mov	bp, ax			; ┓
	mov	ax, ds			; ┣ ES:BP = 串地址
	mov	es, ax			; ┛
	mov	cx, MessageLength	; CX = 串长度
	mov	ax, 01301h		; AH = 13,  AL = 01h
	mov	bx, 0007h		; 页号为0(BH = 0) 黑底白字(BL = 07h)
	mov	dl, 0
	int	10h			; int 10h
	ret


;----------------------------------------------------------------------------
; 函数名: ReadSector
;----------------------------------------------------------------------------
; 作用:
;	从第 ax 个 Sector 开始, 将 cl 个 Sector 读入 es:bx 中
ReadSector:
	; -----------------------------------------------------------------------
	; 怎样由扇区号求扇区在磁盘中的位置 (扇区号 -> 柱面号, 起始扇区, 磁头号)
	; -----------------------------------------------------------------------
	; 设扇区号为 x
	;                           ┌ 柱面号 = y >> 1
	;       x           ┌ 商 y ┤
	; -------------- => ┤      └ 磁头号 = y & 1
	;  每磁道扇区数     │
	;                   └ 余 z => 起始扇区号 = z + 1
	push	bp
	mov	bp, sp
	sub	esp, 2			; 辟出两个字节的堆栈区域保存要读的扇区数: byte [bp-2]

	mov	byte [bp-2], cl
	push	bx			; 保存 bx
	mov	bl, [BPB_SecPerTrk]	; bl: 除数
	div	bl			; y 在 al 中, z 在 ah 中
	inc	ah			; z ++
	mov	cl, ah			; cl <- 起始扇区号
	mov	dh, al			; dh <- y
	shr	al, 1			; y >> 1 (其实是 y/BPB_NumHeads, 这里BPB_NumHeads=2)
	mov	ch, al			; ch <- 柱面号
	and	dh, 1			; dh & 1 = 磁头号
	pop	bx			; 恢复 bx。将 cl 个 Sector 读入 es:bx 中。所以，每次读取数据，需与上次bx所在位置衔接。	
	; 至此, "柱面号, 起始扇区, 磁头号" 全部得到 ^^^^^^^^^^^^^^^^^^^^^^^^
	mov	dl, [BS_DrvNum]		; 驱动器号 (0 表示 A 盘)
.GoOnReading:
	mov	ah, 2			; 读
	mov	al, byte [bp-2]		; 读 al 个扇区
	int	13h
	jc	.GoOnReading		; 如果读取错误 CF 会被置为 1, 这时就不停地读, 直到正确为止

	add	esp, 2
	pop	bp

	ret

;----------------------------------------------------------------------------
; 函数名: GetFATEntry
;----------------------------------------------------------------------------
; 作用:
;	找到序号为 ax 的 Sector 在 FAT 中的条目, 结果放在 ax 中。结果是该文件的剩余部分存储的簇所在序号。
;	需要注意的是, 中间需要读 FAT 的扇区到 es:bx 处, 所以函数一开始保存了 es 和 bx
GetFATEntry:
	push	es
	push	bx
	push	ax
	mov	ax, BaseOfLoader; `.
	sub	ax, 0100h	;  | 在 BaseOfLoader 后面留出 4K 空间用于存放 FAT
	mov	es, ax		; /
	pop	ax
	mov	byte [bOdd], 0
	mov	bx, 3
	mul	bx			; dx:ax = ax * 3
	mov	bx, 2
	div	bx			; dx:ax / 2  ==>  ax <- 商, dx <- 余数
	cmp	dx, 0
	jz	LABEL_EVEN
	mov	byte [bOdd], 1
LABEL_EVEN:;偶数
	; 现在 ax 中是 FATEntry 在 FAT 中的偏移量,下面来
	; 计算 FATEntry 在哪个扇区中(FAT占用不止一个扇区)
	xor	dx, dx			
	mov	bx, [BPB_BytsPerSec]
	div	bx ; dx:ax / BPB_BytsPerSec
		   ;  ax <- 商 (FATEntry 所在的扇区相对于 FAT 的扇区号)
		   ;  dx <- 余数 (FATEntry 在扇区内的偏移)
	push	dx
	mov	bx, 0 ; bx <- 0 于是, es:bx = (BaseOfLoader - 100):00
	add	ax, SectorNoOfFAT1 ; 此句之后的 ax 就是 FATEntry 所在的扇区号
	mov	cl, 2
	call	ReadSector ; 读取 FATEntry 所在的扇区, 一次读两个, 避免在边界
			   ; 发生错误, 因为一个 FATEntry 可能跨越两个扇区
	pop	dx
	add	bx, dx
	mov	ax, [es:bx]
	cmp	byte [bOdd], 1
	jnz	LABEL_EVEN_2
	shr	ax, 4
LABEL_EVEN_2:
	and	ax, 0FFFh

LABEL_GET_FAT_ENRY_OK:

	pop	bx
	pop	es
	ret
;----------------------------------------------------------------------------

times 	510-($-$$)	db	0	; 填充剩下的空间，使生成的二进制代码恰好为512字节
dw 	0xaa55				; 结束标志
```



操作系统、汇编等笔记

https://www.jianshu.com/p/ee2b60719563

