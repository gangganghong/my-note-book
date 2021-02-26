# 写操作系统的笔记



## bochs调试

### bochs断点调试



### bochs使用gdb断点调试

#### 安装bocsh

```shell
./configure --prefix=/home/cg/tools/bochs-2.6.11 --enable-plugins   --enable-x86-64   --enable-cpp  --enable-disasm   --enable-gdb-stub --enable-x86-debugger --enable-e1000 
make
make install
```

Makefile

```makefile
##################################################
# Makefile
##################################################

BOOT:=boot.asm
LDR:=loader.asm
KERNEL:=kernel.asm
BOOT_BIN:=$(subst .asm,.bin,$(BOOT))
LDR_BIN:=$(subst .asm,.bin,$(LDR))
KERNEL_BIN:=$(subst .asm,.bin,$(KERNEL))

IMG:=a.img
FLOPPY:=/mnt/floppy/

.PHONY : everything

everything : $(BOOT_BIN) $(LDR_BIN) $(KERNEL_BIN)
        # ld -s -Ttext 0x30400 -o kernel.bin kernel.o string.o start.o kliba.o -m elf_i386
        ld -Ttext 0x30400 -o kernel.bin kernel.o string.o start.o kliba.o -m elf_i386
        dd if=$(BOOT_BIN) of=$(IMG) bs=512 count=1 conv=notrunc
        sudo mount -o loop $(IMG) $(FLOPPY)
        sudo cp $(LDR_BIN) $(FLOPPY) -v
        sudo cp $(KERNEL_BIN) $(FLOPPY) -v
        sudo umount $(FLOPPY)

clean :
        rm -f $(BOOT_BIN) $(LDR_BIN) $(KERNEL_BIN) *.o

$(BOOT_BIN) : $(BOOT)
        nasm $< -o $@

$(LDR_BIN) : $(LDR)

$(KERNEL_BIN) : $(KERNEL) start.c string.asm
        nasm -f elf -o $(subst .asm,.o,$(KERNEL)) $<
        nasm -f elf -o string.o string.asm
        nasm -f elf -o kliba.o kliba.asm
        gcc -c -fno-builtin -o start.o start.c -g -m32
```

关键语句：

```makefile
# ld 去掉 -s，gcc 加上 -g，编译出来的文件才包含调试信息
ld -Ttext 0x30400 -o kernel.bin kernel.o string.o start.o kliba.o -m elf_i386
gcc -c -fno-builtin -o start.o start.c -g -m32
```



bochsrc

```
###############################################################
# Configuration file for Bochs
###############################################################

# how much memory the emulated machine will have
megs: 32

# filename of ROM images
romimage: file=/usr/local/share/bochs/BIOS-bochs-latest
vgaromimage: file=/usr/local/share/bochs/VGABIOS-lgpl-latest

# what disk images will be used
# floppya: 1_44=freedos.img, status=inserted
# floppyb: 1_44=pm.img, status=inserted
floppya: 1_44="a.img", status=inserted

# choose the boot disk.
boot: a
# boot: floppy
# where do we send log messages?
log: bochsout.txt

# disable the mouse
mouse: enabled=0
#magic_break:enabled=1

gdbstub: enabled=1, port=1234, text_base=0, data_base=0, bss_base=0
# enable key mapping, using US layout as default.

keyboard: keymap=/usr/local/share/bochs/keymaps/x11-pc-us.map

# magic_break:enable=1
```

在C文件中设置断点

```c

```

报错

```shell
start.c: In function 'cstart':
start.c:27:24: error: expected declaration specifiers or '...' before '&' token
         (void *((u32*)(&gdt_ptr[2]))),    /* Base  of Old GDT */
```



调试信息

```shell
(gdb) p p_gdt_base
$16 = (u32 *) 0x32c22 <gdt_ptr+2>
(gdb) p &gdt
$17 = (DESCRIPTOR (*)[128]) 0x32820 <gdt>
(gdb) p gdt_ptr
$18 = "\377\003 (\003"
(gdb) p *gdt_ptr
$19 = 255 '\377'
(gdb) p &gdt_ptr
$20 = (u8 (*)[6]) 0x32c20 <gdt_ptr>
(gdb) x /1nx
Argument required (starting display address).
(gdb) x /1nx 0x32c20
0x32c20 <gdt_ptr>:	0x282003ff
(gdb) p &gdt_ptr[2]
$21 = (u8 *) 0x32c22 <gdt_ptr+2> " (\003"
(gdb) x /1wx 0x32c22
0x32c22 <gdt_ptr+2>:	0x00032820
(gdb) p p_gdt_base
$22 = (u32 *) 0x32c22 <gdt_ptr+2>
```

#### 启动调试

按上面的配置，然后按下面的方式启动调试：

1. 在虚拟机上的命令行工具执行命令`/home/cg/tools/bochs-2.6.11/bin/bochs -f bochsrc-gdb `，如下下图

   1. ![image-20210226162722000](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210226162722000.png)
   2. ![image-20210226162647280](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210226162647280.png)

   3. ![image-20210226162802405](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210226162802405.png)

2. 在物理机的命令行工具执行下面的命令：

   1. ```shell
      [root@localhost f]# gdb ./kernel.bin
      # 用gdb在文件start.c的第31行设置断点
      (gdb) b start.c:31
      Breakpoint 1 at 0x30496: file start.c, line 31.
      # 连接到本地的bochs，1234是在bochsrc中配置的
      (gdb) target remote localhost:1234
      Remote debugging using localhost:1234
      0x0000fff0 in ?? ()
      # 执行到下一个断点
      (gdb) c
      Continuing.
      
      Breakpoint 1, cstart () at start.c:31
      31		u16* p_gdt_limit = (u16*)(&gdt_ptr[0]);
      # 打印变量gdt_ptr
      (gdb) p gdt_ptr
      $1 = "\037\000=\001\t"
      # 打印变量gdt_ptr的内存地址
      (gdb) p &gdt_ptr
      $2 = (u8 (*)[6]) 0x32c20 <gdt_ptr>
      # 执行下一条语句
      (gdb) s
      32		u32* p_gdt_base  = (u32*)(&gdt_ptr[2]);
      (gdb) p p_gdt_limit
      $3 = (u16 *) 0x32c20 <gdt_ptr>
      (gdb) s
      33		*p_gdt_limit = GDT_SIZE * sizeof(DESCRIPTOR) - 1;
      (gdb) p p_gdt_base
      $4 = (u32 *) 0x32c22 <gdt_ptr+2>
      (gdb) p &gdt_ptr[2]
      $5 = (u8 *) 0x32c22 <gdt_ptr+2> "=\001\t"
      (gdb) s
      34		*p_gdt_base  = (u32)&gdt;
      (gdb) s
      35	}
      (gdb) p p_gdt_limit
      $6 = (u16 *) 0x32c20 <gdt_ptr>
      (gdb) p p_gdt_base
      $7 = (u32 *) 0x32c22 <gdt_ptr+2>
      (gdb) p &gdt
      $8 = (DESCRIPTOR (*)[128]) 0x32820 <gdt>
      # 查看内存地址0x32c22中的数据
      (gdb) x /1wx 0x32c22
      0x32c22 <gdt_ptr+2>:	0x00032820
      ```

#### gdb查看内存

用gdb查看内存
格式: x /nfu

参数说明：
x 是 examine 的缩写
n表示要显示的内存单元的个数
f表示显示方式, 可取如下值
x 按十六进制格式显示变量。
d 按十进制格式显示变量。
u 按十进制格式显示无符号整型。
o 按八进制格式显示变量。
t 按二进制格式显示变量。
a 按十六进制格式显示变量。
i 指令地址格式
c 按字符格式显示变量。
f 按浮点数格式显示变量。
u表示一个地址单元的长度
b表示单字节，
h表示双字节，
w表示四字节，
g表示八字节

## 写loader

loader要完成的工作：准备好GDT、进入保护模式、读取内核文件到内存、将内核重新放置到内存中。

实模式下，能使用的最大内存是1M。地址总线是20位，能接收并寻址的最大内存地址是2的20次方-1，内存地址的初始地址是0，所以，能使用的最大内存是1M。

能不能在“准备好GDT、进入保护模式”前完成下面两项工作？

1. 读取内核文件到内存。
2. 将内核重新放置到内存中。

在测试阶段，可以，把内核写得很小就行。实际中，不行。内核所需要的空间必定会大于1M。

于上神在实模式下读取内核文件到内存中。为啥不在进入保护模式后再做这件事情？

没有找到答案，先在进入保护模式前把内核文件读取到内存吧。

怎么把内核文件读取到内存？我可以直接去看“读取loader到内存”的代码。写过的代码，也会忘记。我想先回忆，怎么把内核文件读取到内存。

读取到哪块内存？要确定存储内核文件的内存的初始位置。

用什么方法读取？用bios的系统调用（已经忘记）读取。

### 从软盘中读取数据

从哪里读取？从软盘中读取。（已经忘记了怎么从软盘中读取）。

1. 在FAT12的根目录区通过对比文件名找到目标文件对应的根目录项。
2. 从根目录项中获取目标文件在FAT中的第一个FAT项的编号（初始编号是2）。
3. 目标文件的第一个FAT项的编号的值是第二个FAT项的编号，第二个FAT项的编号是第三个FAT项的编号，剩余的FAT项的编号按照相同的方法查找。FAT项的编号，既是下一个FAT项的编号，又是目标文件在数据区的扇区号。这实质是一个单链表。
   1. FAT项的值，有的表示当前FAT项是最后一个FAT项，有的表示当前FAT项对应的簇是坏簇。具体数值，我忘记了。
4. 每个FAT项占用12位，所以，一个FAT项可能跨越扇区。为了完整获取每个FAT项，需要读取两个扇区。3个字节刚好存储2个FAT项。又忘记了那个计算公式。
5. 两个公式：
   1. 根据扇区在软盘中的扇区号，计算出扇区在软盘中的柱面号、磁头号、在X柱面X磁头的扇区号。
   2. 根据FAT项编号，计算出这个FAT项在软盘中的扇区号。
      1. 一个FAT项占用8个bit，即1.5个字节。FAT项的编号从0开始，若FAT项的编号是X，那么，这个FAT项的偏移量是X个FAT项（0~~~X-1），一共是多少个字节？X * 1.5个字节。在汇编中，X * 1.5 的等价运算是，X * 3，然后，X / 2。
      2. 结果若无余数，刚好占用整数个字节N。结果若有余数，占用整数个字节N + 0.5个字节。
      3. 无论结果是哪种，偏移量都是N个字节，因为，内存寻址的最小单位是字节，而不是其他的（比如BIT）。
      4. 无余数，偏移量就是N个字节。目标FAT项是12个bit，8 bit + 4 bit，从N-1开始，读取1个字节，再读取4个bit。
      5. 有余数，偏移量是N个字节。目标FAT项是12个bit，8 bit + 4 bit。
      6. 在4、5两个步骤想了很久，心算画图都没能解决问题。看代码后，理解了。从N字节（包括）开始，读取2个字节。
         1. 无余数，取两个字节16个bit的低12位。
         2. 有余数，取两个字节16个bit的高12位。因为，低四位属于前一个FAT项。

把数据写入内存，使用的仍然是系统调用（bios还是DOS？），能指定内存位置。

把内核文件写入内存后，再从内存中，把程序头（是这个术语吗）复制到另外的内存位置P，然后将cs:ip指向P，就能将控制权交给内核。

在控制权交给内核前，需要先初始化好GDT、进入保护模式吗？

在内核中，会使用1M之外的内存，因此，需要进入保护模式。在保护模式下，内存寻址需要GDT，因此，需要初始化GDT。

但是，也可以不在内核中使用1M之外的内存吧？

### 重新放置内核到内存

一个星期前，写过这个知识点的文章，此刻，又忘记了。

内核文件，是ELF文件。读取内核到内存，就是把ELF文件的所有程序段读取到指定的内存位置。

ELF文件的构成要素：ELF头，sections(节)，节头表，程序头表。

唉！忘记了，全忘记了！

学习很艰难，或者说，记忆很艰难。我十天前一个字一个字写出的文章，再看，很陌生，好像看不懂了。

使用函数 `memcpy(程序段在内存中的虚拟地址，程序段在内存中的地址，程序段的长度)` 实现功能。复制操作会重复进行，重复次数由程序段的个数决定。

从elf头中能获取由多少个程序段。从程序头中能获取`memcpy`的三个参数。

有几个疑问：

1. 为什么重新放置内核的目标地址是虚拟内存地址？
   1. 不清楚。搁置。后面再看。
2. 虚拟内存地址是什么？
   1. 逻辑地址。段地址：偏移量。
   2. 线性地址。实模式下，段地址*16+偏移量。保护模式下，选择子-->全局描述符-->段基址+偏移量。
   3. 物理地址。没有开启页机制，线性地址就是物理地址。
   4. 虚拟地址。是上面的哪个地址？不是任何一个。涉及到MMU等，在目前的书和代码中，没有提到这个东西，先搁置吧。

#### 实现memcpy

```assembly
; Mempcy(程序段的虚拟内存地址，程序段在内存中的地址，程序段的长度)
; 逐字节复制
; [es:di]--->[ds:si]
Memcpy:
	push ax
	push cx
	push bp
	mov bp, sp
	; 三个参数
	mov cx, [bp+16]
	mov di, [bp+12]
	mov si, [bp+8]
	
.1:
	cmp cx, 0
	jmp .2
	mov al, [ds:si]
	mov [es:di], al
	inc di
	inc si
	dec cx
	jmp .1
	
.2
	mov ax, [bp+8]
	
	pop bp
	pop cx
	pop ax
	
	
	ret
```



很不熟练。大概需要20分钟才能写得出来这个函数。

疑问。函数堆栈中的第一个元素、第二个元素，分别是什么？

![image-20210218173229879](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210218173229879.png)

call的作用是把eip入栈。



![image-20210218173248737](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210218173248737.png)



ret的作用是把参数和eip出栈。



![image-20210218173326540](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210218173326540.png)



长调用，call把cs、eip入栈。

长调用是不同段间的调用。

我不愿先开启保护模式再重新放置内核，是因为我不想写GDT那块代码，更具体地说，我不想写特权级那块的代码。

特权级，我还没有弄明白是咋回事。RPL、CPL、DPL，很烦。

进入保护模式之后，才重新放置内核到内存中。为何如此？因为，在内核中，需要使用1M之上的内存。

#### 开启保护模式

很简单，套路。打开A20，设置cr0。一个跳转。

#### 初始化GDT

这很难。最难的是，设置全局描述符的属性。

需要几个段？至少三个。

第一个：代码段。

第二个：数据段。

第三个：视频段。

上面是我的设想。看看于上神怎么做的。

### v1-loader

第一版v1。实现一个功能：把内核文件读取到内存中。模仿于上神的方法，复制一个新文件夹。

大部分内容与boot一样，从软盘中读取名为“kernel.bin"的文件，并且放置到内存位置为A的那片内存。

我打算直接复制boot.asm的代码。重新写一次，很耗费时间，调试更加耗费时间。这个知识，并不常用。

但是，可以复述一下读取数据到内存的过程。

使用bios的系统调用读取数据。这个系统调用需要一些参数，柱面号、磁头号、偏移扇区号，读取到哪里。

#### 用扇区号计算bios系统调用的参数

根据扇区号，能计算出所需要的参数。扇区号又能从软盘的头信息中获取。

扇区号除以每磁道包含的扇区数，商是磁道数，余数是在X柱面X磁头的扇区偏移量，扇区号是余数+1。

磁道数除以每个柱面包含的磁道的数量，商是柱面号，余数没有作用。

磁道数 & 1，结果是磁头号。（这个公式，理解起来，仍然有点费劲）

这个计算方法的前提是，每个柱面包含2个磁道，有两个磁头。

#### 用FAT编号计算扇区号

1. 使用FAT编号计算出字节偏移量O、整数字节偏移量ON。
   1. O是整数，O等于ON，字节O、字节O+1两个字节的低12位是当前FAT项。
   2. O不是整数，字节ON、字节ON+1两个字节的高12位是当前FAT项。因为低4位上一个FAT项的高4位。
2. 判断FAT编号为N的FAT项的字节偏移量是不是整数字节的方法：
   1. N * 3 / 2
   2. 检查上一步的余数是不是0
      1. 是0，整数。
      2. 不是0，非整数。
   3. 原理：一个FAT项占用1.5个字节，编号为N的FAT项的字节偏移量是N * 1.5。上面的计算公式是N * 1.5的等价运算。

怎么写loader？完全没有思路。只能参考于神的代码。

没有section，开头必须是`org 0x100`。

#### debug1

读取内核文件的时候，获取到的第一个FAT项的编号是0。这是不正确的。

我试图验证，是读取kernel文件时出错了吗？

已知di，如何查看某个段内的数据？用bochs。

````shell
xp /1wx 0x800:0x100

xp /1wx 0x800:0x120
xp /1wx 0x800:0x140
xp /1wx 0x800:0x160
xp /1wx 0x800:0x180

xp /1wx 0x800:0x19a

<bochs:16> r
CPU0:
rax: 00000000_0000004c
rbx: 00000000_00000b90
rcx: 00000000_0009000b
rdx: 00000000_00000000
rsp: 00000000_0000ffd4
rbp: 00000000_00000000
rsi: 00000000_000e7d07
rdi: 00000000_00000100
r8 : 00000000_00000000
r9 : 00000000_00000000
r10: 00000000_00000000
r11: 00000000_00000000
r12: 00000000_00000000
r13: 00000000_00000000
r14: 00000000_00000000
r15: 00000000_00000000
rip: 00000000_00007c78


xp /1wx 0x800:0x17a

xp /1bx 0x0:0xe7d06


# si 的值虽然是 00000000_000e7d07，但是，lodsb 的等价运算 lodsb al, byte ptr ds:[si] 中的 si 的值应该是 0x7d06 而不是 0xe7d06。原因未知，这是运行代码得到的结果。

xp /1bx 0x0:0x7d06

xp /1bx 0x9000:0xe7d06


xxd -u -a -g 1 -c 16 -s +9728 -l 1024 a.img
````

### v2-loader

#### 重新放置内核

在实模式下，`0x80000`是能够被使用的内存地址。`0x80000`是32KB，在1MB之内。

elf格式的内核文件的开头是elf头。从elf头中，能获取：

1. 程序头的个数。
2. 程序头表在内核文件中的偏移量。

内核文件的物理位置，加上程序头表在内核文件中的偏移量，等于程序头表的位置。在第一个程序头中，能获取这个程序头对应的程序段的虚拟内存地址、目标程序段在文件中的偏移量、目标程序段的长度。

怎么从elf头和程序头中获取数据呢？根据元素的数据类型。假如，在elf头中，程序头的个数是第三个元素，前两个元素的数据类型都占用4个字节，那么，程序头的个数从第8个字节开始。具体数字，我忘记了。

#### Memcpy

按字节复制，第三个参数是多少，就循环多少次。每次循环，把数据从第二个参数内存位置复制到第一个参数内存位置。

使用什么完成的复制？用al做中间变量。在[es:di]和[ds:si]之间复制。es和ds的值，需要注意。

这个函数有返回值。返回值是第一个参数内存位置，把它放在ax中返回。

循环，仍然不使用loop，而是使用cx手工完成。

调用函数时，`call Memcpy`，需要对`sp`进行操作吗？是增还是减？数量呢？

`sub sp, 4`。使用减。是必须的。原因是，不能影响与函数调用无关的栈。

~~例如，函数参数入栈前，sp的位置是16，`sub sp,4`后，sp的位置是12，入栈一个参数后，sp的位置是10。位置10到位置0的数据不会覆盖位置16。这段解说是错误的。~~

只需要在调用结束后，对sp进行加操作，释放参数占用的栈空间。

可以用 `/home/cg/yuyuan-os/osfs09/f` 中的文件分析elf、FAT12系统。

#### 分析elf

```shell
[root@localhost v2]# xxd -u -a -g 1 -c 16  kernel.bin
00000000: 7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
00000010: 02 00 03 00 01 00 00 00 00 04 03 00 34 00 00 00  ............4...
00000020: 18 04 00 00 00 00 00 00 34 00 20 00 01 00 28 00  ........4. ...(.
00000030: 03 00 02 00 01 00 00 00 00 00 00 00 00 00 03 00  ................
00000040: 00 00 03 00 06 04 00 00 06 04 00 00 05 00 00 00  ................
00000050: 00 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000060: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
*
00000400: EB FE EB FE EB FE 00 2E 73 68 73 74 72 74 61 62  ........shstrtab
00000410: 00 2E 74 65 78 74 00 00 00 00 00 00 00 00 00 00  ..text..........
00000420: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000430: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000440: 0B 00 00 00 01 00 00 00 06 00 00 00 00 04 03 00  ................
00000450: 00 04 00 00 06 00 00 00 00 00 00 00 00 00 00 00  ................
00000460: 10 00 00 00 00 00 00 00 01 00 00 00 03 00 00 00  ................
00000470: 00 00 00 00 00 00 00 00 06 04 00 00 11 00 00 00  ................
00000480: 00 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00  ................
```



```shell
00000040: 00 00 03 00 06 04 00 00 06 04 00 00 05 00 00 00  ................
```



```shell
00000400: EB FE EB FE EB FE 00 2E 73 68 73 74 72 74 61 62  ........shstrtab
```



`EB FE EB FE EB FE`是下面代码的机器码，总计6个字节，与`00000040: 00 00 03 00 06 04 00 00 06 04 00 00 05 00 00 00  ................`中的`06`吻合。

```assembly
[section .text]

global _start

_start:
;       mov ax, 0xB800
        jmp $
        jmp $
        jmp $
;       mov gs, ax
;       mov ah, 0Fh
;       mov al, 'C'
;       mov [gs:(80 * 20 + 40) * 2], ax
;       jmp $
```



仅存的疑问：

```assembly
; InitKernel ---------------------------------------------------------------------------------
; 将 KERNEL.BIN 的内容经过整理对齐后放到新的位置
; 遍历每一个 Program Header，根据 Program Header 中的信息来确定把什么放进内存，放到什么位置，以及放多少。
; --------------------------------------------------------------------------------------------
InitKernel:
        xor   esi, esi
        mov   cx, word [BaseOfKernelFilePhyAddr+2Ch];`. ecx <- pELFHdr->e_phnum
        movzx ecx, cx                               ;/
        mov   esi, [BaseOfKernelFilePhyAddr + 1Ch]  ; esi <- pELFHdr->e_phoff
        add   esi, BaseOfKernelFilePhyAddr;esi<-OffsetOfKernel+pELFHdr->e_phoff
.Begin:
        mov   eax, [esi + 0]
        cmp   eax, 0                      ; PT_NULL
        jz    .NoAction
        push  dword [esi + 010h]    ;size ;`.
        mov   eax, [esi + 04h]            ; |
        add   eax, BaseOfKernelFilePhyAddr; | memcpy((void*)(pPHdr->p_vaddr),
        push  eax		    ;src  ; |      uchCode + pPHdr->p_offset,
        push  dword [esi + 08h]     ;dst  ; |      pPHdr->p_filesz;
        call  MemCpy                      ; |
        add   esp, 12                     ;/
.NoAction:
        add   esi, 020h                   ; esi += pELFHdr->e_phentsize
        dec   ecx
        jnz   .Begin

        ret
```



```assembly
mov   eax, [esi + 04h]            ; |
add   eax, BaseOfKernelFilePhyAddr; | memcpy((void*)(pPHdr->p_vaddr),
```



`BaseOfKernelFilePhyAddr`是文件开头，内容是：`00000000: 7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............`。

可是，我猜想，重新放置内核，需要复制的内容是：`00000400: EB FE EB FE EB FE 00 2E 73 68 73 74 72 74 61 62  ........shstrtab`。没有找到能达到这个目的的代码。

#### 小结

今天，又耗费了一天，没有完成“重新放置内存”。

1. 直接读取elf文件到内存中，就像处理DOS文件一样，跳转到elf文件，就能执行。那么，就没有必要使用elf了。
2. 理解不了于上神重新放置内存的代码，具体说，源代码的位置，理解不了。
3. 直接操作显存，显存数值的列与行坐标，都不能超过最大值，否则，数据不会在屏幕上显示。
4. 解析elf文件、fat12系统，一定要分析一个实例，并且把分析过程准确记录下来。若心算太慢，就用笔算。
5. 不能读取全部指令，那么，这些指令一条也不会被执行。



为啥进展这么慢？时间跨度长达一天，我不记得准确细节了。遇到棘手的问题，立刻用写作来辅助思路，不要心算。

#### debug

```shell
prefetch: EIP [00010000] > CS.limit [0000ffff]
```

.COM文件的大小为什么不能超过64KB？

在实模式下跳转

```shell
loader.asm:192: warning: word data exceeds bounds [-w+number-overflow]
loader.asm:192: warning: word data exceeds bounds [-w+number-overflow]
```



```shell
<bochs:2> sreg
es:0x0000, dh=0x00009300, dl=0x0000ffff, valid=7
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
cs:0xf000, dh=0xff0093ff, dl=0x0000ffff, valid=7
	Data segment, base=0xffff0000, limit=0x0000ffff, Read/Write, Accessed
ss:0x0000, dh=0x00009300, dl=0x0000ffff, valid=7
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
ds:0x0000, dh=0x00009300, dl=0x0000ffff, valid=7
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
fs:0x0000, dh=0x00009300, dl=0x0000ffff, valid=7
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
gs:0x0000, dh=0x00009300, dl=0x0000ffff, valid=7
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0000, dh=0x00008200, dl=0x0000ffff, valid=1
tr:0x0000, dh=0x00008b00, dl=0x0000ffff, valid=1
gdtr:base=0x0000000000000000, limit=0xffff
idtr:base=0x0000000000000000, limit=0xffff
```



#### 实例分析elf文件一

```assembly
[section .text]
STR:    db      "Hello"
LEN     equ     $-STR

global _start

_start:
        jmp $
        mov eax, 4

        mov ebx, 1
        mov ecx, STR
        mov edx, LEN
        int 0x80
        mov eax, 1
        mov ebx, 0
        int 0x80
```



用下面的命令编译这个文件

```shell
[root@localhost v2]# nasm -f elf -o kernel.o kernel.asm
# 在 64 位 centos 上编译32位汇编代码，需要使用 -m elf_i386
[root@localhost v2]# ld -s -o kernel kernel.o -m elf_i386
```

查看编译出来的文件kernel的二进制数据

```shell
[root@localhost v2]# xxd -u -a -g 1 -c 16 kernel
00000000: 7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
00000010: 02 00 03 00 01 00 00 00 65 80 04 08 34 00 00 00  ........e...4...
00000020: 98 00 00 00 00 00 00 00 34 00 20 00 01 00 28 00  ........4. ...(.
00000030: 03 00 02 00 01 00 00 00 00 00 00 00 00 80 04 08  ................
00000040: 00 80 04 08 87 00 00 00 87 00 00 00 05 00 00 00  ................
00000050: 00 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000060: 48 65 6C 6C 6F B8 04 00 00 00 BB 01 00 00 00 B9  Hello...........
00000070: 60 80 04 08 BA 05 00 00 00 CD 80 B8 01 00 00 00  `...............
00000080: BB 00 00 00 00 CD 80 00 2E 73 68 73 74 72 74 61  .........shstrta
00000090: 62 00 2E 74 65 78 74 00 00 00 00 00 00 00 00 00  b..text.........
000000a0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
000000b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
000000c0: 0B 00 00 00 01 00 00 00 06 00 00 00 60 80 04 08  ............`...
000000d0: 60 00 00 00 27 00 00 00 00 00 00 00 00 00 00 00  `...'...........
000000e0: 10 00 00 00 00 00 00 00 01 00 00 00 03 00 00 00  ................
000000f0: 00 00 00 00 00 00 00 00 87 00 00 00 11 00 00 00  ................
00000100: 00 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00  ................
```



第一个程序头

![image-20210221140113344](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210221140113344.png)

第一个程序段在文件中的偏移量

![image-20210221140214796](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210221140214796.png)

第一个程序段的虚拟内存地址`08048000`。

![image-20210221140427497](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210221140427497.png)

第一个程序段的长度：0087H

![image-20210221140533376](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210221140533376.png)

第一个程序段的内容：

![image-20210221140658787](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210221140658787.png)

程序的入口：`08048065`。

![image-20210221143530667](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210221143530667.png)

第一个程序段被复制到的虚拟内存地址是`08048000h`，程序的入口也是一个虚拟内存地址：`08048065h`。可见，前面的`65h`个字节不是指令，换句话说，从第`66h`个字节开始，才是指令数据。这段数据有多长呢？第一个程序段的长度是`0087H`，所以，第一个程序段的数据是：

```shell
[root@localhost v2]# xxd -u -a -g 1 -c 16 kernel
00000000: 7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
00000010: 02 00 03 00 01 00 00 00 65 80 04 08 34 00 00 00  ........e...4...
00000020: 9C 00 00 00 00 00 00 00 34 00 20 00 01 00 28 00  ........4. ...(.
00000030: 03 00 02 00 01 00 00 00 00 00 00 00 00 80 04 08  ................
00000040: 00 80 04 08 89 00 00 00 89 00 00 00 05 00 00 00  ................
00000050: 00 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000060: 48 65 6C 6C 6F EB FE B8 04 00 00 00 BB 01 00 00  Hello...........
00000070: 00 B9 60 80 04 08 BA 05 00 00 00 CD 80 B8 01 00  ..`.............
00000080: 00 00 BB 00 00 00 00
```

第一个程序段的指令数据的开始位置是距离文件偏移量为`65h`个字节的位置，即

```shell
00000060: 48 65 6C 6C 6F EB FE B8 04 00 00 00 BB 01 00 00  Hello...........
00000070: 00 B9 60 80 04 08 BA 05 00 00 00 CD 80 B8 01 00  ..`.............
00000080: 00 00 BB 00 00 00 00
```

把指令数据单独截取出来，是

```shell
# ----------------- 表示非指令数据
00000060: ----------------- EB FE B8 04 00 00 00 BB 01 00 00  Hello...........
00000070: 00 B9 60 80 04 08 BA 05 00 00 00 CD 80 B8 01 00  ..`.............
00000080: 00 00 BB 00 00 00 00
```

`jmp $`的二进制代码是`EB FE`，吻合。再验证一次。

将源代码修改为：

```assembly
[section .text]
STR:    db      "Hello"
LEN     equ     $-STR

global _start

_start:
        jmp $
        mov eax, 4

        mov ebx, 1
        mov ecx, STR
        mov edx, LEN
        int 0x80
        mov eax, 1
        mov ebx, 0
        int 0x80
        jmp $
```



```shell
[root@localhost v2]# xxd -u -a -g 1 -c 16 kernel
00000000: 7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
00000010: 02 00 03 00 01 00 00 00 65 80 04 08 34 00 00 00  ........e...4...
00000020: 9C 00 00 00 00 00 00 00 34 00 20 00 01 00 28 00  ........4. ...(.
00000030: 03 00 02 00 01 00 00 00 00 00 00 00 00 80 04 08  ................
00000040: 00 80 04 08 8B 00 00 00 8B 00 00 00 05 00 00 00  ................
00000050: 00 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000060: 48 65 6C 6C 6F EB FE B8 04 00 00 00 BB 01 00 00  Hello...........
00000070: 00 B9 60 80 04 08 BA 05 00 00 00 CD 80 B8 01 00  ..`.............
00000080: 00 00 BB 00 00 00 00 CD 80 EB FE 00 2E 73 68 73  .............shs
00000090: 74 72 74 61 62 00 2E 74 65 78 74 00 00 00 00 00  trtab..text.....
000000a0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
000000b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
000000c0: 00 00 00 00 0B 00 00 00 01 00 00 00 06 00 00 00  ................
000000d0: 60 80 04 08 60 00 00 00 2B 00 00 00 00 00 00 00  `...`...+.......
000000e0: 00 00 00 00 10 00 00 00 00 00 00 00 01 00 00 00  ................
000000f0: 03 00 00 00 00 00 00 00 00 00 00 00 8B 00 00 00  ................
00000100: 11 00 00 00 00 00 00 00 00 00 00 00 01 00 00 00  ................
00000110: 00 00 00 00
```



程序入口虚拟内存地址：`65 80 04 08`，`08048065h`。它在文件中的偏移量是24个字节。

第一个程序段的长度是：`8B 00 00 00`，`8Bh`。它在程序头中的偏移量是`10h`个字节。二进制数据如下：

```shell
00000000: 7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
00000010: 02 00 03 00 01 00 00 00 65 80 04 08 34 00 00 00  ........e...4...
00000020: 9C 00 00 00 00 00 00 00 34 00 20 00 01 00 28 00  ........4. ...(.
00000030: 03 00 02 00 01 00 00 00 00 00 00 00 00 80 04 08  ................
00000040: 00 80 04 08 8B 00 00 00 8B 00 00 00 05 00 00 00  ................
00000050: 00 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000060: 48 65 6C 6C 6F EB FE B8 04 00 00 00 BB 01 00 00  Hello...........
00000070: 00 B9 60 80 04 08 BA 05 00 00 00 CD 80 B8 01 00  ..`.............
00000080: 00 00 BB 00 00 00 00 CD 80 EB FE
```

第一个程序段的虚拟内存地址是：`00 80 04 08`，`08048000h`。它在程序头中的偏移量是`08h`个字节。

上面二进制数据中的行号如`00000010`是十六进制数，是数据在文件中的位置编号（内存地址是内存中的位置编号）。把这些数据复制到内存中后，如果能查看内存中的数据，这些数据的内存地址的编号的相对位置（偏移量）和它们在文件中的相对位置相同。如果把这个程序段复制到内存地址为`08048000h`初始位置的一段内存，那么，在内存中，二进制数据如下：

```shell
08048000: 7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
08048010: 02 00 03 00 01 00 00 00 65 80 04 08 34 00 00 00  ........e...4...
08048020: 9C 00 00 00 00 00 00 00 34 00 20 00 01 00 28 00  ........4. ...(.
08048030: 03 00 02 00 01 00 00 00 00 00 00 00 00 80 04 08  ................
08048040: 00 80 04 08 8B 00 00 00 8B 00 00 00 05 00 00 00  ................
08048050: 00 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
08048060: 48 65 6C 6C 6F EB FE B8 04 00 00 00 BB 01 00 00  Hello...........
08048070: 00 B9 60 80 04 08 BA 05 00 00 00 CD 80 B8 01 00  ..`.............
08048080: 00 00 BB 00 00 00 00 CD 80 EB FE
```

二进制数据的入口地址是`08048065h`，很容易得到下面的结论。

指令二进制数据如下：

```shell
00000060: ----------- EB FE B8 04 00 00 00 BB 01 00 00  Hello...........
00000070: 00 B9 60 80 04 08 BA 05 00 00 00 CD 80 B8 01 00  ..`.............
00000080: 00 00 BB 00 00 00 00 CD 80 EB FE
```

开头和结尾都是`EB FE`，源代码的开头和结尾也都是`jmp $`，吻合。

#### 疑难杂症

1. .COM文件的大小为啥不能超过64KB？
2. 在loader中跳转到kernel，跳转目标是随意设定的，但仍然能执行kernel中的指令，这是为什么？kernel是elf文件、DOS文件都是如此。
3. 内核有必要编译成ELF文件吗？
4. 由于可随意指定跳转的目标地址，所以，无法验证重新放置内核是否正确。怎么验证重新放置内核是否正确？



昨天就被这个问题阻碍了，一直没有明确问题，而是在分析ELF文件等。没有详细记录，总之，时间被不明不白地消耗掉了。



```shell
prefetch: EIP [00010000] > CS.limit [0000ffff]
```

哪条语句导致这个问题？

没有`jmp $`会导致这句。	

一个重要的猜想：cpu执行机器指令时，若当前指令是合法指令，就执行；若是不能识别的指令，就跳过继续执行下一条指令。

基于这个猜想，上面的第2个问题，就能解释清楚了。无论kernel是ELF文件还是DOS文件或其他文件，将IP指向这个文件，遇到了文件中的合法指令，就会执行。因此，我能看到执行效果。

既然如此，似乎没有必要在内存中重新放置ELF文件啊，把它们全部复制到内存中，然后逐条执行不也可以吗？的确是可以，我能想到的弊端除了“耗费内存、耗费CPU”外，再也想不到其他需要ELF的必要性了。

我已经验证了上面的猜想。验证方法是：

1. 用两条`jmp $`作为kernel的结尾。第2条kernel的偏移量是`73h`。那么，跳过所有指令的地址相对于第一个段的开头的偏移量是`75h`。

2. 将跳转目标设置为：`jmp BaseOfKernel:75h`。执行结果，没有打印字符，重复出现`prefetch: EIP [00010000] > CS.limit [0000ffff]`。

3. 将跳转目标设置为：`jmp BaseOfKernel:73h`。没有打印字符，也没有重复出现`prefetch: EIP [00010000] > CS.limit [0000ffff]`。这是执行了`jmp $`的原因。

4. 将跳转目标设置为：`jmp BaseOfKernel:61h`。执行结果，打印出了字符。

   1. ip应该指向程序的入口地址，据此，跳转目标应该如此设置：`jmp BaseOfKernel:60h`。可是执行结果却有多远的闪烁字符。

      1. ```shell
         ext at t=14396751
         (0) [0x000000080060] 8000:0060 (unk. ctxt): mov eax, 0xe88eb800       ; 66b800b88ee8
         ```

      2. ```
         <bochs:3> s
         Next at t=14396751
         (0) [0x000000080061] 8000:0061 (unk. ctxt): mov ax, 0xb800            ; b800b8
         ; 66 b800 b8 8ee8
         ; 	 b800 b8
         ```

      3. ```shell
         00000060: 66 B8 00 B8 8E E8 B4 0F B0 43 65 66 A3 D0 0C 00  f........Cef....
         ```

      4. 跳转目标的偏移量应该是`60h`。可执行结果为何异常？

下面是用来验证的源代码：

```assembly
;[section .text]

;global _start

;_start:
        ;mov ax, 2
        ;jmp $
        mov ax, 0xB800
       ;jmp $
       ;jmp $
       ;jmp $
        mov gs, ax
        mov ah, 0Fh
        mov al, 'C'
        mov [gs:(80 * 20 + 40) * 2], ax
        jmp $
        jmp $
```



编译上面的代码后，用xxd查看二进制数据：

```assembly
[root@localhost v2]# xxd -u -a -g 1 -c 16 kernel.bin
00000000: 7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
00000010: 02 00 03 00 01 00 00 00 60 80 04 08 34 00 00 00  ........`...4...
00000020: 88 00 00 00 00 00 00 00 34 00 20 00 01 00 28 00  ........4. ...(.
00000030: 03 00 02 00 01 00 00 00 00 00 00 00 00 80 04 08  ................
00000040: 00 80 04 08 75 00 00 00 75 00 00 00 05 00 00 00  ....u...u.......
00000050: 00 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000060: 66 B8 00 B8 8E E8 B4 0F B0 43 65 66 A3 D0 0C 00  f........Cef....
00000070: 00 EB FE EB FE 00 2E 73 68 73 74 72 74 61 62 00  .......shstrtab.
00000080: 2E 74 65 78 74 00 00 00 00 00 00 00 00 00 00 00  .text...........
00000090: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
000000a0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
000000b0: 0B 00 00 00 01 00 00 00 06 00 00 00 60 80 04 08  ............`...
000000c0: 60 00 00 00 15 00 00 00 00 00 00 00 00 00 00 00  `...............
000000d0: 10 00 00 00 00 00 00 00 01 00 00 00 03 00 00 00  ................
000000e0: 00 00 00 00 00 00 00 00 75 00 00 00 11 00 00 00  ........u.......
000000f0: 00 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00  ................
```



从上面的二进制数据中提取关键数据：

1. 第一个程序段的长度：

   1. ```shell
      00000030: ----------- 01 00 00 00 00 00 00 00 00 80 04 08  ................
      00000040: 00 80 04 08 75 00 00 00 75 00 00 00 05 00 00 00  ....u...u.......
      00000050: 00 10 00 00 -----------------------------------   ................
      ```

   2. ```
      00000030: ----------- ** ** ** ** ** ** ** ** ** ** ** **  ................
      00000040: ** ** ** ** 75 00 00 00 ** ** ** ** ** ** ** **  ....u...u.......
      00000050: ** ** ** ** -----------------------------------   ................
      ```

   3. 经过上面的分析，结果是`75 00 00 00`，`75h`。

2. 第一个程序段在文件中的偏移量：`00 80 04 08`

3. 程序的入口地址（即ip应该指向哪个地址)：

   1. ```shell
      00000000: 7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
      00000010: 02 00 03 00 01 00 00 00 60 80 04 08 34 00 00 00  ........`...4...
      ```

   2. 偏移量是：`60 80 04 08`，它在文件中的偏移量是`24h`。

##### 验证重新放置后的内核

下面的代码都运行在实模式下。

通过bochs断点调试验证。

```assembly
; Memcpy(p_vaddr, p_off, p_size)
Memcpy:
        push bp
        ; 在新元素入栈前获取sp，方便获取函数的参数。
        mov bp, sp
        push ax
        push cx
        push si
        push di
        ;mov bp, sp
        ;mov di, [bp + 4]        ; p_vaddr，即 dst
        ;mov si, [bp + 8]        ; p_off，即 src
        ;mov cx, [bp + 12]      ; 程序头的个数，即p_size

        ;mov di, [bp + 8]        ; p_vaddr，即 dst
        ;mov si, [bp + 12]        ; p_off，即 src
        ;mov cx, [bp + 16]       ; 程序头的个数，即p_size
				
				; 增幅为何是2？因为，这是在实模式下的汇编代码，每个参数的长度是16位，即2个字节。
				; 注意，[bp+4]中的4的单位是字节，8个bit。
				; 为什么第一个参数是[bp+4]？因为开头push bp，[bp+2]是原始bp。
				; 那么，[bp+0]是什么？[bp+0]是call Func入栈的ip。
        mov di, [bp + 4]        ; p_vaddr，即 dst。函数的第一个参数。
        mov si, [bp + 6]        ; p_off，即 src。函数的第二个参数。
        mov cx, [bp + 8]       ; 程序头的个数，即p_size。函数的第三个参数。
        ; 因为，这个函数的功能是，把数据从0x8000开始的内存复制到从0x6000开始的内存，所以，要修改es。
        ; 因为下面复制数据是在[es:di]和[ds:si]之间复制。
        push es
        mov es, di
        mov di, 0

.1:
        mov byte al, [ds:si]
        mov [es:di], al
        inc si
        inc di
        dec cx

        cmp cx, 0
        jz .2
        jmp .1

.2:
        pop es
        mov ax, [bp + 4]

        pop di
        pop si
        pop cx
        pop ax
        pop bp

        ret
```



```assembly
; 重新放置内核
InitKernel:
       push ax
       push cx
       push si
       ; commont start
       ; mov cx, [BaseOfKernel3 + 2CH] 的实质是 mov cx, word ptr ds:0x802c这种语句，所以，需要修改ds和BaseOfKernel3。
       mov ax, 0x8000
       push ds
       mov ds, ax
       ; commont end
        xchg bx, bx
       ;程序段的个数
        ;mov cx, word ptr ds:0x802c
       ; mov cx, [BaseOfKernel3 + 2CH] 等价于 mov cx, word ptr ds:BaseOfKernel3 + 2CH。
       ; 这是断点调试看到的，原因未知。
       ; 另外，mov cx, word ptr 0x0:0x802c 结果与 mov cx, word ptr 0x8000:0x2c 不同。
       mov cx, [BaseOfKernel3 + 2CH]
       ; mov ax, [BaseOfKernel + 2CH]
       ;程序头表的内存地址
       xor si, si
       mov si, [BaseOfKernel3 + 1CH]
       add si, BaseOfKernel3
        xchg bx, bx

.Begin2:
      mov ax, [si + 10H]
      ;push word [si + 10H]
      push word ax

       mov ax, BaseOfKernel3
       add ax, [si + 4H]
       push word ax
        mov ax, [si + 8H]
        push ax
        ;push word [si + 8H]
        xchg bx, bx
        call Memcpy
        xchg bx, bx
        ; 三个参数，占用3个字,6个字节
        add sp, 6
        dec cx
        cmp cx, 0
        jz .NoAction
        add si, 20H
        jmp .Begin2
;
.NoAction:
        pop ds
        pop si
        pop cx
        pop ax

        ret
        
BaseOfKernel    equ     0x8000
BaseOfKernel2   equ     0x6000
BaseOfKernel3   equ     0x0
OffSetOfLoader  equ     0x0
BaseOfFATEntry  equ     0x1000
```



耗费了两天左右（可能是三天），才写完了实模式下的“重新放置内核”功能。大概在昨天这个时候取得突破。

遇到的问题分别有：

1. Loader.asm中的代码不执行。boot.asm只能读取一个扇区的数据。偶然想到的，之前遇到过。
2. Kernel.asm执行不正常（没有打印应该打印的字符）。设置显存坐标超过最大值了，例如，把行坐标设置为28。
3. kernel是不是elf文件都能执行。我的猜测，cpu遇到可执行指令就执行，遇到不可执行指令要么什么也不做要么报错。只要设置的起点在可执行指令的前面，总会执行到可执行指令。因此，无论是否elf文件都能执行。
4. 不重新放置内核都能正常执行。原因同3。
5. 栈
   1. 实模式下栈的增幅是2个字节（16个bit），我把增幅写成了4个字节（32个bit）。
   2. 在函数调用中，先将原始bp入栈，再通过bp从栈中获取参数。
6. 验证“重新放置内核“是否成功。例如，将内核重新放置到了0x6000:0x400，那么，通过bochs断点调试来查看二进制代码与xxd在查看kernel.bin到的二进制代码是否一致就能验证”重新放置内核“是否成功。
   1. 这个验证，只能证明只有一个程序段的内核被重新放置。



解决这个问题为什么很慢？没有明确问题是什么，仅仅只是觉得不正确，然后就一次次地没有明确目的地调试。突破出现在我明确界定问题之后。

`kernel.asm`内容如下：

```assembly
[section .text]

global _start

_start:
        ;mov ax, 2
        ;jmp $
        xchg bx, bx
        mov ax, 0xB800
       ;jmp $
       ;jmp $
       ;jmp $
        mov gs, ax
        mov ah, 0Fh
        mov al, 'C'
        mov [gs:(80 * 20 + 40) * 2], ax
        mov [gs:(80 * 21 + 40) * 2], ax
        mov [gs:(80 * 22 + 80) * 2], ax
        jmp $
        jmp $
```

上面的源代码中是否包含“global _start _start:"等语句不影响这份源代码能不能编译成elf文件。只要在编译时指定了`-f elf`就能把它编译成elf文件。我的猜想，`section`等没有对应的二进制代码，只是给人类看的，在二进制文件中不包含`section`这类指令。

#### bochs xp

x /nuf [addr] 显示线性地址的内容

xp /nuf [addr] 显示物理地址的内容

n 显示的单元数

u 每个显示单元的大小，u可以是下列之一：

b BYTE

h WORD

w DWORD

g DWORD64

注意: 这种命名法是按照GDB习惯的，而并不是按照inter的规范。

### v3-loader

这个版本，完成下面的功能：

1. 准备好GDT。
2. 开启保护模式。
3. 进入保护模式。
4. 重新放置内核。



最大的难点是，准备好GDT，更具体地说，是特权级(CPL、DPL、RPL)。

完全不知道怎么写。想到什么写什么吧。

重新放置内核是在实模式下还是在保护模式下？在保护模式下。因为，内核的入口地址是`0x30400`。在下面的代码中，

```assembly
READ_FILE_OVER:
        xchg bx, bx
        mov al, 'O'
        mov ah, 0Dh
        mov [gs:(80 * 23 + 33) * 2], ax
        ; 在内存中重新放置内核
        call InitKernel

        xchg bx, bx
        ;jmp BaseOfKernel:73h
        ;jmp BaseOfKernel:61h
        jmp BaseOfKernel2:400h
        ;jmp BaseOfKernel:60h
        ;jmp BaseOfKernel:0
        ;jmp BaseOfKernel:OffSetOfLoader
        ;jmp BaseOfKernel2:0x30400
        ;jmp BaseOfKernel:OffSetOfLoader
        ;jmp BaseOfKernel:40h
        ;jmp OVER
        
BaseOfKernel2   equ     0x30000  
```

`jmp BaseOfKernel2:400h`会跳转到`0x30400`。

编译时出错：

```shell
[root@localhost v3]# make
nasm -o loader.bin loader.asm
loader.asm:194: warning: word data exceeds bounds [-w+number-overflow]
dd if=boot.bin of=a.img count=1 conv=notrunc
```



```shell
<bochs:2> sreg
es:0x0000, dh=0x00009300, dl=0x0000ffff, valid=7
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
cs:0xf000, dh=0xff0093ff, dl=0x0000ffff, valid=7
	Data segment, base=0xffff0000, limit=0x0000ffff, Read/Write, Accessed
ss:0x0000, dh=0x00009300, dl=0x0000ffff, valid=7
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
ds:0x0000, dh=0x00009300, dl=0x0000ffff, valid=7
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
fs:0x0000, dh=0x00009300, dl=0x0000ffff, valid=7
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
gs:0x0000, dh=0x00009300, dl=0x0000ffff, valid=7
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0000, dh=0x00008200, dl=0x0000ffff, valid=1
tr:0x0000, dh=0x00008b00, dl=0x0000ffff, valid=1
gdtr:base=0x0000000000000000, limit=0xffff
idtr:base=0x0000000000000000, limit=0xffff
```



~~我的猜想是，在实模式下，`jmp 段地址：偏移量`中的段地址的最大值是`0xffff`。`jmp 0x30000`超过了这个最大值，因此不能正确执行。~~

16位寄存器不能存储`0x30000`，这才是这份代码不能正确执行的原因。

#### 需要设置几个全局描述符

不知道。

在哪里需要用到全局描述符？不知道。

在哪里需要用到GDT的选择子。不知道。

全局描述符的属性应该怎么设置？不知道。这些属性不能随意设置吗？不知道。

这么多不知道的知识点，空想也没有用，只能看别人的代码了。一直以来，我都是这样做的。

怎么写操作系统？我遵循的流程是：

1. 自己想。
2. 想不出怎么写，看别人的代码，理解每一行代码，记下来，能复述出来。
3. 根据复述出来的知识点，自己动手写代码。
4. 测试代码。

以后的每项功能，都用这个流程去开发。



#### 实模式下的内存寻址方式

在实模式下，内存的寻址方式是：

1. 内存地址是：段地址:偏移量，例如：`0x6000:0x400`。
2. 这个内存地址的物理地址是：`0x60400`。计算公式是：物理地址 = 段地址 * 16 + 偏移量。

怎么验证这个公式呢？用下面的方法。

```assembly
jmp BaseOfKernel2:400h
BaseOfKernel2:	db 0x6000
```

使用bochs断点查看

```shell
<bochs:8> xp /1hx 0x6000:0x400
[bochs]:
0x0000000000060400 <bogus+       0>:	0x8766
<bochs:9> xp /1hx 0x60400
[bochs]:
0x0000000000060400 <bogus+       0>:	0x8766
```

内存`0x6000:0x400`和内存`0x60400`中的数据相同，这证明上面的公式是正确的。

在保护模式下，内存地址仍然是”段：偏移量“。不过，”段“是”全局描述符的选择子，而“偏移量”变成了物理地址。仍然不是非常理解这种表示方法。

全局描述符的选择子，表示为：全局描述符的标号 - 第一个全局描述符。

首先，建立全局描述符宏H，这个宏有三个参数，分别是：段基址、段界限、段属性。段属性最难。

然后，用H创建全局描述符。暂时只创建这几个描述符：空描述符、显存描述符。

加载GDT，使用`lgdt [GDTPtr]`。`GdtPtr`是啥？它的结构是：GDT界限 GDT的初始地址。

有一个寄存器，名称是`gdtr`，专门用来存储GDT的物理地址。gdtr的结构如下：

```shell
-----------------------------------------------------------
|			32位基地址													|		16位界限				|
-----------------------------------------------------------
```

#### 好多疑问

1. ```assembly
   ; GDT ------------------------------------------------------------------------------------------------------------------------------------------------------------
   ;                                                段基址            段界限     , 属性
   LABEL_GDT:			Descriptor             0,                    0, 0						; 空描述符
   LABEL_DESC_FLAT_C:		Descriptor             0,              0fffffh, DA_CR  | DA_32 | DA_LIMIT_4K			; 0 ~ 4G
   LABEL_DESC_FLAT_RW:		Descriptor             0,              0fffffh, DA_DRW | DA_32 | DA_LIMIT_4K			; 0 ~ 4G
   LABEL_DESC_VIDEO:		Descriptor	 0B8000h,               0ffffh, DA_DRW                         | DA_DPL3	; 显存首地址
   ; GDT ------------------------------------------------------------------------------------------------------------------------------------------------------------
   ```

   1. 设置了`LABEL_DESC_FLAT_C`、`LABEL_DESC_FLAT_RW` 这两个段基址和段界限相同、属性不同的段。这两个段的数据会互相覆盖吗？数据能共用吗？
   2. 例如，在`LABEL_DESC_FLAT_C:0x2`存储数据`aa`，那么，通过`LABEL_DESC_FLAT_RW:0x2`读取到的数据也是`aa`吗？
   3. 在`LABEL_DESC_FLAT_C:0x2`存储数据`aa`，在`LABEL_DESC_FLAT_RW:0x2`存储数据`bb`，那么，通过`LABEL_DESC_FLAT_C:0x2`和`LABEL_DESC_FLAT_RW:0x2`读取到的数据都是`bb`吗？



#### debug

```assembly
; GDT ------------------------------------------------------------------------------------------------------------------------------------------------------------
;                                                段基址            段界限     , 属性
LABEL_GDT:			Descriptor             0,                    0, 0						; 空描述符
LABEL_DESC_FLAT_C:		Descriptor             0,              0fffffh, DA_CR  | DA_32 | DA_LIMIT_4K			; 0 ~ 4G
LABEL_DESC_FLAT_RW:		Descriptor             0,              0fffffh, DA_DRW | DA_32 | DA_LIMIT_4K			; 0 ~ 4G
LABEL_DESC_VIDEO:		Descriptor	 0B8000h,               0ffffh, DA_DRW                         | DA_DPL3	; 显存首地址
; GDT ------------------------------------------------------------------------------------------------------------------------------------------------------------

GdtLen		equ	$ - LABEL_GDT
GdtPtr		dw	GdtLen - 1				; 段界限
		dd	BaseOfLoaderPhyAddr + LABEL_GDT		; 基地址

; GDT 选择子 ----------------------------------------------------------------------------------
SelectorFlatC		equ	LABEL_DESC_FLAT_C	- LABEL_GDT
SelectorFlatRW		equ	LABEL_DESC_FLAT_RW	- LABEL_GDT
SelectorVideo		equ	LABEL_DESC_VIDEO	- LABEL_GDT + SA_RPL3
; GDT 选择子 ----------------------------------------------------------------------------------

jmp	dword SelectorFlatRW:(BaseOfLoaderPhyAddr+LABEL_PM_START)
```



引发下面的错误：



```shell
<bochs:3> s
00016183765e[CPU0  ] check_cs(0x0010): not a valid code segment !
00016183765e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x0d)
00016183765e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x08)
00016183765i[CPU0  ] CPU is in protected mode (active)
00016183765i[CPU0  ] CS.mode = 16 bit
00016183765i[CPU0  ] SS.mode = 16 bit
00016183765i[CPU0  ] EFER   = 0x00000000
00016183765i[CPU0  ] | EAX=60000011  EBX=00000007  ECX=00000009  EDX=534d0400
00016183765i[CPU0  ] | ESP=00000100  EBP=000002a7  ESI=000e029d  EDI=0000007a
00016183765i[CPU0  ] | IOPL=0 id vip vif ac vm RF nt of df if tf sf zf af PF cf
00016183765i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
00016183765i[CPU0  ] |  CS:9000( 0004| 0|  0) 00090000 0000ffff 0 0
00016183765i[CPU0  ] |  DS:9000( 0005| 0|  0) 00090000 0000ffff 0 0
00016183765i[CPU0  ] |  SS:9000( 0005| 0|  0) 00090000 0000ffff 0 0
00016183765i[CPU0  ] |  ES:9000( 0005| 0|  0) 00090000 0000ffff 0 0
00016183765i[CPU0  ] |  FS:0000( 0005| 0|  0) 00000000 0000ffff 0 0
00016183765i[CPU0  ] |  GS:0000( 0005| 0|  0) 00000000 0000ffff 0 0
00016183765i[CPU0  ] | EIP=00000281 (00000281)
00016183765i[CPU0  ] | CR0=0x60000011 CR2=0x00000000
00016183765i[CPU0  ] | CR3=0x00000000 CR4=0x00000000
(0).[16183765] [0x000000090281] 9000:0000000000000281 (unk. ctxt): jmpf 0x0010:00090380      ; 66ea800309001000
00016183765e[CPU0  ] exception(): 3rd (13) exception with no resolution, shutdown status is 00h, resetting
00016183765i[SYS   ] bx_pc_system_c::Reset(HARDWARE) called
00016183765i[CPU0  ] cpu hardware reset
00016183765i[APIC0 ] allocate APIC id=0 (MMIO enabled) to 0x0000fee00000
00016183765i[CPU0  ] CPU[0] is the bootstrap processor
00016183765i[CPU0  ] CPUID[0x00000000]: 00000005 68747541 444d4163 69746e65
00016183765i[CPU0  ] CPUID[0x00000001]: 00000633 00010800 00002028 17cbfbff
00016183765i[CPU0  ] CPUID[0x00000002]: 00000000 00000000 00000000 00000000
00016183765i[CPU0  ] CPUID[0x00000003]: 00000000 00000000 00000000 00000000
00016183765i[CPU0  ] CPUID[0x00000004]: 00000000 00000000 00000000 00000000
00016183765i[CPU0  ] CPUID[0x00000005]: 00000040 00000040 00000003 00000020
00016183765i[CPU0  ] CPUID[0x80000000]: 80000008 68747541 444d4163 69746e65
00016183765i[CPU0  ] CPUID[0x80000001]: 00000633 00000000 00000101 ebd3f3ff
00016183765i[CPU0  ] CPUID[0x80000002]: 20444d41 6c687441 74286e6f 7020296d
00016183765i[CPU0  ] CPUID[0x80000003]: 65636f72 726f7373 00000000 00000000
00016183765i[CPU0  ] CPUID[0x80000004]: 00000000 00000000 00000000 00000000
00016183765i[CPU0  ] CPUID[0x80000005]: 01ff01ff 01ff01ff 40020140 40020140
00016183765i[CPU0  ] CPUID[0x80000006]: 00000000 42004200 02008140 00000000
00016183765i[CPU0  ] CPUID[0x80000007]: 00000000 00000000 00000000 00000000
00016183765i[CPU0  ] CPUID[0x80000008]: 00003028 00000000 00000000 00000000
```

这个错误是段属性造成的。

`jmp	dword SelectorFlatC:(BaseOfLoaderPhyAddr+LABEL_PM_START)` 会把`cs`设置成`SelectorFlatC`的值`0x0010`。例如：

```assembly
; GDT ------------------------------------------------------------------------------------------------------------------------------------------------------------
;                                                段基址            段界限     , 属性
LABEL_GDT:			Descriptor             0,                    0, 0						; 空描述符
LABEL_DESC_FLAT_C:		Descriptor             0,              0fffffh, DA_CR  | DA_32 | DA_LIMIT_4K			; 0 ~ 4G
LABEL_DESC_FLAT_RW:		Descriptor             0,              0fffffh, DA_CR | DA_32 | DA_LIMIT_4K			; 0 ~ 4G
LABEL_DESC_VIDEO:		Descriptor	 0B8000h,               0ffffh, DA_DRW                         | DA_DPL3	; 显存首地址
; GDT ------------------------------------------------------------------------------------------------------------------------------------------------------------

GdtLen		equ	$ - LABEL_GDT
GdtPtr		dw	GdtLen - 1				; 段界限
		dd	BaseOfLoaderPhyAddr + LABEL_GDT		; 基地址

; GDT 选择子 ----------------------------------------------------------------------------------
SelectorFlatC		equ	LABEL_DESC_FLAT_C	- LABEL_GDT
SelectorFlatRW		equ	LABEL_DESC_FLAT_RW	- LABEL_GDT
SelectorVideo		equ	LABEL_DESC_VIDEO	- LABEL_GDT + SA_RPL3
; GDT 选择子 ----------------------------------------------------------------------------------

jmp	dword SelectorFlatRW:(BaseOfLoaderPhyAddr+LABEL_PM_START)
```



```shell
(0) [0x000000090281] 9000:0000000000000281 (unk. ctxt): jmpf 0x0010:00090380      ; 66ea800309001000
<bochs:2> sreg
es:0x9000, dh=0x00009309, dl=0x0000ffff, valid=1
	Data segment, base=0x00090000, limit=0x0000ffff, Read/Write, Accessed
cs:0x9000, dh=0x00009309, dl=0x0000ffff, valid=1
	Data segment, base=0x00090000, limit=0x0000ffff, Read/Write, Accessed
ss:0x9000, dh=0x00009309, dl=0x0000ffff, valid=7
	Data segment, base=0x00090000, limit=0x0000ffff, Read/Write, Accessed
ds:0x9000, dh=0x00009309, dl=0x0000ffff, valid=3
	Data segment, base=0x00090000, limit=0x0000ffff, Read/Write, Accessed
fs:0x0000, dh=0x00009300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
gs:0x0000, dh=0x00009300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0000, dh=0x00008200, dl=0x0000ffff, valid=1
tr:0x0000, dh=0x00008b00, dl=0x0000ffff, valid=1
gdtr:base=0x000000000009013d, limit=0x1f
idtr:base=0x0000000000000000, limit=0x3ff
<bochs:3> s
Next at t=16183766
(0) [0x000000090380] 0010:0000000000090380 (unk. ctxt): xchg bx, bx               ; 6687db
<bochs:4> sreg
es:0x9000, dh=0x00009309, dl=0x0000ffff, valid=1
	Data segment, base=0x00090000, limit=0x0000ffff, Read/Write, Accessed
cs:0x0010, dh=0x00cf9b00, dl=0x0000ffff, valid=1
	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
ss:0x9000, dh=0x00009309, dl=0x0000ffff, valid=7
	Data segment, base=0x00090000, limit=0x0000ffff, Read/Write, Accessed
ds:0x9000, dh=0x00009309, dl=0x0000ffff, valid=3
	Data segment, base=0x00090000, limit=0x0000ffff, Read/Write, Accessed
fs:0x0000, dh=0x00009300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
gs:0x0000, dh=0x00009300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0000, dh=0x00008200, dl=0x0000ffff, valid=1
tr:0x0000, dh=0x00008b00, dl=0x0000ffff, valid=1
gdtr:base=0x000000000009013d, limit=0x1f
idtr:base=0x0000000000000000, limit=0x3ff
```

在bochs中查看内存中的数据，仍然使用“段地址：偏移量”，例如：`xp /1wx 0x0010:0x90380`，bochs会自动计算出事件内存地址并且打印出该内存中的数据。在实模式下，我还能手工直接算出内存的实际地址，在保护模式下，计算实际内存地址很麻烦。

纠正上面的说法，只是从选择子计算对应的段基址比较麻烦，实际内存地址（线性地址）的计算方法仍然是：段基址 + 偏移量。下面的断点调试结果能够验证这个结论。

```shell
<bochs:18> xp /1wx 0x8:0x90380
[bochs]:
0x0000000000090380 <bogus+       0>:	0x66db8766
<bochs:19> xp /1wx 0x90380
[bochs]:
0x0000000000090380 <bogus+       0>:	0x66db8766
```

当前语境中，`0x8`对应的段的段基址是0。

若是如此，我有很充分的理由相信：不同的段，段基址相同，段偏移量相同，那么，这两个段的内存空间是重合的。

```shell
<bochs:6> xp /1wx 0x0008:0x00000000000905da
[bochs]:
0x00000000000905da <bogus+       0>:	0x00c3d975
<bochs:7> xp /1wx 0x00000000000905da
[bochs]:
0x00000000000905da <bogus+       0>:	0x00c3d975
<bochs:8> xp /1wx 0x0010:0x00000000000905da
[bochs]:
0x00000000000905da <bogus+       0>:	0x00c3d975
<bochs:9> 
```

上面的断点数据，又证明了不同的段（段基址、段界限）使用的相同的内存空间。那么，有啥必要弄两个基本相同、只是属性不同的段？

内存访问为啥要分段访问？

1. 为了重定位。以前，程序中使用内存的物理地址，要同时运行多个程序非常麻烦（需要在程序中写死它所使用的内存，还要保证这片内存不被其他程序改写）。
2. 为了访问所有内存。以前，能访问的内存很小，要访问全部内存，只能将内存划分为段基址不同的若干个段，然后通过在不同的段之间跳转来访问所有的内存。
3. 后来，能够用一个段访问全部内存空间，为啥还是要分段访问内存？



#### 段描述符的属性

详情见《操作系统真相还原》4.3.1节。

在这个问题上耗费了40多分钟，原因是：两本书上的说明不一致。

《操作系统真相还原》

![image-20210223152935498](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210223152935498.png)



《X86汇编语言：从实模式到保护模式》

![image-20210223153134430](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210223153134430.png)

上图中的红色注释是错误的，应该是：第0位是A，其他的依次是，R(第1位)、C（第2位）、X（第3位）。

《一个操作系统的实现》

![image-20210223153212938](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210223153212938.png)

三本书都没有错。第一本书和后面两本书对应不上，是因为第一本书没有按照第0位到第3位的顺序排列。

对代码段来说，按照从第0位到第3位的顺序排列，依次是：X、C、R、A。用下面的实例来理解：

1. Ch。12 = 8 + 4。1100。X:1，C:1。可执行、一致代码段。
2. Ah。10 = 8 + 2。1010。X:1,C:0,R:1。可执行、可读。
3. Bh。11 = 8 + 3。1011。X：1，C：0，R：1，A：1。可执行、可读、已访问。

TYPE和S结合起来才有意义。S为0时，表示系统段；S为1时，表示数据段（又叫非系统段）。

在第一张图中，非系统段又分为代码段和数据段。怎么区分二者呢？看X位是多少。X是1，代码段。X是0，数据段。代码段可执行，数据段不可执行。只用一位，自然不能同时表示这个段可执行、又不可执行。这或许就是于上神要对0~~4GB这片内存创建两个段描述符的原因吧。感觉这个解释仍然不是很让我信得过。

数据段一定是可读的，一定是不可执行的。

代码段一定是不可写的，一定是可执行的。

创建段描述符的宏的第三个参数，段属性，写成多个值组成的比较好。段属性的构成要素是：TYPE、S、DPL、P、AVL、L、D/B、G。我觉得，直接把所有的要素都写到参数中更好，直观。如果只写出一个结果，例如'Ch'，可读性太差，还需要计算。

目前，可执行段、可读写段、视频段：

1. 都是非系统段，S为1。
2. DPL
   1. 可执行段、可读写段，设置为0。
   2. 视频段，设置为：3。必须为3，经过运行，确实如此。原因未知。
3. P，设置为0。
4. AVL，设置为0。
5. L，设置为0。
6. D/B，设置为1。
7. G，设置为1。
8. TYPE
   1. 可执行段，设置为：X：1，C：1，R：0，A：0。Ah。
   2. 可读写段，设置为：X：0，E：0，W：1，A：0。2h。
   3. 视频段，设置为：X：0，E：0，W：1，A：0。2h。

一直深感恐惧的段描述符及其属性，在这个阶段，只需要简单设置就行了。

#### 别人的代码流程

1. FAT12文件头。

2. 用宏创建三个段描述符。

   1. 可执行段。
   2. 可读写段。
   3. 视频段。

3. 创建GdtPtr。它存储GDT的物理地址。GDT的物理地址就是GDT中的第一个元素的地址。

   1. 本结构由48个bit组成。低16位是界限，高32位是基地址。
   2. 基地址是GDT中的第一个元素的物理地址。
   3. 界限是GDT的最后一个字节的地址，等于GDT的长度-1，因为，计算从0开始。例如，长度是3，从0开始计数，最后一个地址是2。
   4. CPU提供了专门存储GDT物理地址的寄存器，`lgdt`。加载GDT物理地址到`lgdt`的语法：`lgdt [GdtPtr]`。

4. 创建段选择子。

   1. 很容易。用目标描述符的标号-第一个描述符的标号。
   2. 选择子的结构：第0~2位是RPL，第3位是T1，第4~15位是描述符索引（GDT中的偏移量，第几个描述符）。
   3. T1是0，表示这个选择子是在GDT中索引段描述符；T1是1，表示这个选择子是在LDT中索引段描述符。
   4. 为什么要创建选择子？在保护模式下，段地址：偏移量的寻址方式中，段地址是段的选择子。创建选择子，是保护模式下内存寻址的需要。

5. 进入保护模式。

   1. 上面准备好GDT和GDT的物理地址后，下面开启保护模式。

   2. 关中断，`cli`。在进入保护模式的过程中，CPU处理中断的机制发生改变，若接收到中断（实模式下的中断协议）会发生错误。我的猜想：跳入32位代码之前，中断机制就在某个步骤发生了改变。

   3. 打开A20。

      1. 就是往一个端口写入数据，使用`in`、`out`指令。

      2. 哪个端口？92h。把端口92h的第2位设置为1。

      3. 代码：

         1. ```assembly
            ; 把端口92h中的数据读入al
            in al, 92h
            ; 把al中的数据的第2位设置为1
            or al, 10h
            ; 把修改过后的al中的数据写到端口92h中
            out 92h, al
            ```

         2. `in`、`out`两个指令的操作数的顺序，很容易混淆。这样记忆吧，这两个指令等同于`mov`，第一个参数都是数据的目标位置。

   4. 设置cr0的PE位（cr0的第0位）为1。

      1. PE是0时，CPU处于实模式；PE是1时，CPU处于保护模式。

      2. 代码：

         1. ```assembly
            mov eax, cr0
            or eax, 1
            mov cr0, eax
            ```

         2. 为啥能直接用eax？不是还没有进入保护模式吗？我的猜想：打开了A20，就已经能使用大于20位的数据线了。

         3. 还不清楚A20的工作机制。

   5. 把GDT的物理地址加载到`lgdt`，`lgdt [GdtPtr]`。

   6. 跳转到32位代码。注意，这个跳转需要指定数据类型（？这个说法准确吗），`jmp dword`。

   7. 上面的步骤能改变顺序吗？

6. 进入保护模式后，cs被`jmp`设置成了可执行代码段，然后，显式设置ds、es、fs、ss的值为可读可写代码段选择子，设置gs的值是视频段的选择子。

7. 在内存中重新放置内核。

8. 跳转到内核的第一个程序段的入口。

#### 创建gdt

十多天写过创建GDT的宏，可我又忘记了。

先用C语言写出全局描述符的结构：

```c
struct descriptor{
  unsigned short SegmentLimitLow;
  unsigned short SegmentBaseLow;
  unsigned char	SegmentBaseMid;
  unsigned char SegmentAttributeLow;
  unsigned char SegmentBaseLimitHigh_AttributeHigh;
  unsigned char SegmentBaseHigh;
};
```

用nasm汇编写全局描述符的结构。

```assembly
; 三个参数分别是段基址、段界限、段属性
; 分别用 %1、%2、%3表示上面的三个参数
%macro	Descriptor 3
	dw	%2 & ffffh
	dw	%1 & ffffh
	db	(%1 >> 16) & ffh
	db	%3 & ffh
	db	((%2 >> 16) & fh) | (((%3 >> 8) & fh) << 4)
	db	(%1 >> 24) & ffh
%endmacro
```



可执行段的属性：

 1. TYPE：X--1，C--0，R--0，A--0

 2. S：0

 3. DPL：0

 4. D/B：1

 5. G：1，

 6. 按照段描述符中的属性位置排列起来：

     1. | G    | D/B  | L    | AVL  | P    | DPL  | DPL  | S    | A    | R    | C    | X    |
        | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
        | 1    | 1    | 0    | 0    | 0    | 0    | 0    | 0    | 0    | 0    | 0    | 1    |
        |      |      |      |      |      |      |      |      |      |      |      |      |

    2. 表格写反了，应该是：

       | X    | C    | R    | A    | S    | DPL  | DPL  | P    | AVL  | L    | D/B  | G    |
       | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
       | 1    | 0    | 0    | 0    | 1    | 0    | 0    | 1    | 0    | 0    | 1    | 1    |

    3. 上面的表格错了，应该是：

       | A    | R    | C    | X    | S    | DPL  | DPL  | P    | AVL  | L    | D/B  | G    |
       | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
       | 0    | 1    | 0    | 1    | 1    | 0    | 0    | 1    | 0    | 0    | 1    | 1    |

可读写段：

| X    | E    | W    | A    | S    | DPL  | DPL  | P    | AVL  | L    | D/B  | G    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 0    | 0    | 1    | 0    | 1    | 0    | 0    | 1    | 0    | 0    | 1    | 1    |

视频段：

| X    | E    | W    | A    | S    | DPL  | DPL  | P    | AVL  | L    | D/B  | G    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 0    | 0    | 1    | 0    | 1    | 1    | 1    | 1    | 0    | 0    | 0    | 0    |



可读写段：

| A    | W    | E    | X    | S    | DPL  | DPL  | P    | AVL  | L    | D/B  | G    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 0    | 1    | 0    | 0    | 1    | 0    | 0    | 1    | 0    | 0    | 1    | 1    |

`0493h`。应该是`0c92h`。

视频段：

| A    | W    | E    | X    | S    | DPL  | DPL  | P    | AVL  | L    | D/B  | G    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 0    | 1    | 0    | 0    | 1    | 1    | 1    | 1    | 0    | 0    | 0    | 0    |

`04f0h`

视频段：

| A    | W    | E    | X    | S    | DPL  | DPL  | P    | AVL  | L    | D/B  | G    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 0    | 0    | 0    | 0    | 1    | 1    | 1    | 1    | 0    | 0    | 1    | 0    |

`0f2h`

视频段：

| A    | W    | E    | X    | S    | DPL  | DPL  | P    | AVL  | L    | D/B  | G    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 0    | 1    | 0    | 0    | 1    | 1    | 1    | 1    | 0    | 0    | 0    | 0    |

`04f0h`。应该是：`0f4h`。应该是`0f2h`。

使用宏创建GDT：

```assembly
LABEL_GDT:	0,												0,																	0
LABLE_GDT_FLAT_X:	0,									fffffh,															893h
LABLE_GDT_FLAT_WR:	0,								fffffh,															293h
LABLE_GDT_VIDEO:		b800h,						ffffh,															2f0h
```

创建选择子：

```assembly
GdtLen	equ		$ - LABEL_GDT
GdtPtr	dw	GdtLen - 1
				dd	BaseOfLoader * 10 + LABEL_GDT
SelectFlatX	equ	LABLE_GDT_FLAT_X - LABEL_GDT
SelectFlatWR	equ	LABLE_GDT_FLAT_WR - LABEL_GDT
SelectVideo		equ	LABLE_GDT_VIDEO - LABEL_GDT + 3
```

##### debug

`jmp dword SelectFlatX:(BaseOfLoader * 10h + LABEL_PM_START)`报错：

```shell
(0) [0x00000009023e] 9000:000000000000023e (unk. ctxt): jmpf 0x0008:00090248      ; 66ea480209000800
<bochs:13> s
00014803608e[CPU0  ] check_cs(0x0008): not a valid code segment !
00014803608e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x0d)
00014803608e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x08)
00014803608i[CPU0  ] CPU is in protected mode (active)
00014803608i[CPU0  ] CS.mode = 16 bit
00014803608i[CPU0  ] SS.mode = 16 bit
00014803608i[CPU0  ] EFER   = 0x00000000
00014803608i[CPU0  ] | EAX=60000011  EBX=00000600  ECX=00090002  EDX=0000000a
00014803608i[CPU0  ] | ESP=0000ffce  EBP=00000000  ESI=000e007c  EDI=0000007a
00014803608i[CPU0  ] | IOPL=0 id vip vif ac vm RF nt of df if tf sf zf af PF cf
00014803608i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
00014803608i[CPU0  ] |  CS:9000( 0004| 0|  0) 00090000 0000ffff 0 0
00014803608i[CPU0  ] |  DS:9000( 0005| 0|  0) 00090000 0000ffff 0 0
00014803608i[CPU0  ] |  SS:0000( 0005| 0|  0) 00000000 0000ffff 0 0
00014803608i[CPU0  ] |  ES:8000( 0005| 0|  0) 00080000 0000ffff 0 0
00014803608i[CPU0  ] |  FS:0000( 0005| 0|  0) 00000000 0000ffff 0 0
00014803608i[CPU0  ] |  GS:b800( 0005| 0|  0) 000b8000 0000ffff 0 0
```

调试了很久。目前，仍不知道问题在哪里。

先看看选择子为`0x0008`的段描述符：

```
xp /1gx 0x000000000009013e + 8
xp /1gx 0x0000000000090146
xp /1gx 0x0000000000090146 +8
xp /1gx 0x000000000009014e
xp /1gx 0x000000000009014e + 8
xp /1gx 0x0000000000090156

<bochs:6> xp /1gx 0x000000000009013e
[bochs]:
0x000000000009013e <bogus+       0>:	0x00000000
<bochs:7> xp /1gx 0x0000000000090146
[bochs]:
0x0000000000090146 <bogus+       0>:	0x8f93000000ffff
<bochs:8> xp /1gx 0x000000000009014e
[bochs]:
0x000000000009014e <bogus+       0>:	0xffffdb87db87
<bochs:9> xp /1gx 0x0000000000090156
[bochs]:
0x0000000000090156 <bogus+       0>:	0x8000ffff002f9300

10001111100100110000000000000000000000001111111111111111							; 56位，可执行段
111111111111111111011011100001111101101110000111											; 48位，可读写段
1000000000000000111111111111111100000000001011111001001100000000			; 64位，视频段

1011 1000000000000000				; b8000h
1111111111111111						; 0ffffh
10 1111 0000									; 2f0


100000 11110000  00001011   10000000000000001111111111111111

; 52 位，视频段，少了12位

00000000         111111110000         


111111110000        ; 只有 12 位，还差四位。在段描述符中，尤其是中间位数，不能缺少

00001011    
10000000000000001111111111111111	; 40位，还差24位	
10000000000000001111111111111111



111100000000101110000000000000001111111111111111
11110000  00001011    10000000000000001111111111111111

0010   0000   11110000   00001011    10000000000000001111111111111111
111100100000101110000000000000001111111111111111
111100100000101110000000000000001111111111111111
111100100000101110000000000000001111111111111111
11110010  00001011     10000000000000001111111111111111


0
0FFFFFh:		1111 1111 1111 1111 1111
398h:				0011 1001	1000
  398h:				1100 1001	0001
c91h

				00000000
11110010            00001011         10000000000000001111111111111111
								00000000000000001111111111111111

							00000000
0011 1111   10011000					00000000      00000000000000001111111111111111
											00000000000000001111111111111111
				     00000000				     00000000			
1100     1111     1001 1011     00000000				00000000000000001111111111111111	
																00000000000000001111111111111111

10001111100100110000000000000000000000001111111111111111								; 56位
1000111110010011   00000000      00000000000000001111111111111111
							00000000
11 1111 10011000    00000000				00000000000000001111111111111111
		 									00000000000000001111111111111111
												
```



```shell
Couldn't open log file: bochsout.txt, using stderr instead
```

执行bochs的用户无权限向bochsout.txt写入数据，导致上面的问题。修改bochsout.txt的用户是bochs的执行用户后，解决问题。使用命令：`chown cg bochsout.txt`。

下面的问题：

```shell
00014033826i[BIOS  ] Booting from 0000:7c00
268 00014803875e[CPU0  ] load_seg_reg(ES, 0x0fff): invalid segment
269 00014803875e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x0d)
270 00014803875e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x08)
271 00014803875i[CPU0  ] CPU is in protected mode (active)
272 00014803875i[CPU0  ] CS.mode = 16 bit
273 00014803875i[CPU0  ] SS.mode = 16 bit
274 00014803875i[CPU0  ] EFER   = 0x00000000
275 00014803875i[CPU0  ] | EAX=60000ad1  EBX=00000600  ECX=00090002  EDX=00000057
276 00014803875i[CPU0  ] | ESP=0000ffce  EBP=00000000  ESI=000e007c  EDI=0000007a
277 00014803875i[CPU0  ] | IOPL=0 id vip vif ac vm RF nt of df if tf sf zf af pf CF
278 00014803875i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
279 00014803875i[CPU0  ] |  CS:0010( 0002| 0|  0) 00000000 000fffff 0 0
280 00014803875i[CPU0  ] |  DS:0018( 0003| 0|  0) 00000000 000fffff 0 0
281 00014803875i[CPU0  ] |  SS:0018( 0003| 0|  0) 00000000 000fffff 0 0
282 00014803875i[CPU0  ] |  ES:0018( 0003| 0|  0) 00000000 000fffff 0 0
283 00014803875i[CPU0  ] |  FS:0018( 0003| 0|  0) 00000000 000fffff 0 0
284 00014803875i[CPU0  ] |  GS:000b( 0001| 0|  3) 000b8000 0000ffff 0 0
285 00014803875i[CPU0  ] | EIP=00000460 (00000460)
286 00014803875i[CPU0  ] | CR0=0x60000011 CR2=0x00000000
287 00014803875i[CPU0  ] | CR3=0x00000000 CR4=0x00000000
288 00014803875e[CPU0  ] exception(): 3rd (13) exception with no resolution, shutdown status is 00h, resetting
289 00014803875i[SYS   ] bx_pc_system_c::Reset(HARDWARE) called
```



```shell
278 00014803875i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
279 00014803875i[CPU0  ] |  CS:0010( 0002| 0|  0) 00000000 000fffff 0 0
280 00014803875i[CPU0  ] |  DS:0018( 0003| 0|  0) 00000000 000fffff 0 0
281 00014803875i[CPU0  ] |  SS:0018( 0003| 0|  0) 00000000 000fffff 0 0
282 00014803875i[CPU0  ] |  ES:0018( 0003| 0|  0) 00000000 000fffff 0 0
283 00014803875i[CPU0  ] |  FS:0018( 0003| 0|  0) 00000000 000fffff 0 0
284 00014803875i[CPU0  ] |  GS:000b( 0001| 0|  3) 000b8000 0000ffff 0 0
```



怎么理解？



```shell
xp /1gx 0x000000000009013d
xp /1gx 0x0000000000090145

<bochs:11> sreg
es:0x0010, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
cs:0x0008, dh=0x00cf9b00, dl=0x0000ffff, valid=1
	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
ss:0x0010, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
ds:0x0010, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
fs:0x0010, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
gs:0x001b, dh=0x0000f30b, dl=0x8000ffff, valid=1
	Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0000, dh=0x00008200, dl=0x0000ffff, valid=1
tr:0x0000, dh=0x00008b00, dl=0x0000ffff, valid=1
gdtr:base=0x000000000009013d, limit=0x1f
idtr:base=0x0000000000000000, limit=0x3ff
<bochs:12> xp /1gx 0x000000000009013d
[bochs]:
0x000000000009013d <bogus+       0>:	0x00000000
<bochs:13> xp /1gx 0x0000000000090145
[bochs]:
0x0000000000090145 <bogus+       0>:	0xcf9b000000ffff

```



```shell
00014033826i[BIOS  ] Booting from 0000:7c00
     268 00014351127e[CPU0  ] write_virtual_checks(): write beyond limit, r/w
     269 00014351135e[CPU0  ] write_virtual_checks(): write beyond limit, r/w
     270 00014351143e[CPU0  ] write_virtual_checks(): write beyond limit, r/w
     271 00014351151e[CPU0  ] write_virtual_checks(): write beyond limit, r/w
```



整个白天都耗费在进入保护模式以后。在进入保护模式前，选择子的标号中的数据都是正常的。进入保护模式后，选择子的标号中的数据发生了改变。我反复尝试，在网上找了些资料，都没能解决问题。

选择子在不同模式下发生改变，ds、gs的值不能被正常设置。原因不明。

今天还做了一件事情：描述符的属性。耗费了太多太多时间。怎么弄正确的呢？从于上神的代码中断点获取描述符的属性，结合书本推测出应该怎样获取正确的属性。

描述符的属性，应该是下面这个表格：

| A    | R    | C    | X    | S    | DPL  | DPL  | P    | AVL  | L    | D/B  | G    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 0    | 1    | 0    | 1    | 1    | 0    | 0    | 1    | 0    | 0    | 1    | 1    |

视频段：

| A    | W    | E    | X    | S    | DPL  | DPL  | P    | AVL  | L    | D/B  | G    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 0    | 1    | 0    | 0    | 1    | 1    | 1    | 1    | 0    | 0    | 0    | 0    |

`04f0h`。应该是：`0f4h`。应该是`0f2h`。

可读写段：

| A    | W    | E    | X    | S    | DPL  | DPL  | P    | AVL  | L    | D/B  | G    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 0    | 1    | 0    | 0    | 1    | 0    | 0    | 1    | 0    | 0    | 1    | 1    |

`0493h`。应该是`0c92h`。

左边是低位，右边是高位。将数据填写到上面的表格后，把数据倒序，然后四个一组分为三组，把每组都转换成十六进制数，把这些数字连起来，就是最终的属性值。

为什么慢？遇到问题，没有先想明确思路，靠直觉重复调试，思考得太少。

这还是只是开始，睡前很高兴，觉得有了重大突破，哪知道醒来就被问题卡住了。后面不知道还有多少个问题等着我解决。像这样慢，不知道哪天才能做完。

另外，睡前，明明看到正确的结果了，却未及时存档，导致现在无法再写出正确的代码。



读取loader时，为啥要设置偏移？

设置了偏移后，在开启保护模式后，为啥获取loader中的变量时不加上偏移量而只加上物理地址？

```assembly
; 真正进入保护模式
	jmp	dword SelectorFlatC:(BaseOfLoaderPhyAddr+LABEL_PM_START)
```



在实模式下，物理地址等于段地址*16+偏移量。在上面的代码中，`BaseOfLoaderPhyAddr`这个名字不恰当，让人误解。它并不是物理地址，而是段基址乘以16的结果。应该把`BaseOfLoaderPhyAddr+LABEL_PM_START`当作一个整体看待。

在真正进入保护模式前，段寄存器cs的值还未改变，此时，代码中的一个标号的偏移量的计算方法是实模式下物理地址的计算方法。

这个理解，我总觉得有点牵强（或者说，有说不过去的地方），不过，目前我只能这样理解，不知道其他更好的理解方法。

#### 进入保护模式debug

所有关键代码，应该都没有问题。能够正确进入32位代码段，可是，无法正确改变ds、gs的值。

出现报错信息：

```shell
269 00014803609e[CPU0  ] write_virtual_checks(): write beyond limit, r/w
270 00014803609e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x0d)
271 00014803609e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x08)
272 00014803609i[CPU0  ] CPU is in protected mode (active)
```

表面原因是：进入32位代码后，选择子的值变化了，不再是实模式下取得的正确的值，然后将这些值设置给gs、ds时导致错误。

这个问题，由于我无脑调试，耗费了11个小时却毫无进展，手都按疼了。

##### 怎么办

1. 描述符有没有问题？
2. 编译时是不是应该设置`[BITS 32]`这些？
3. 再重新对比一次我的代码和正确的代码。无收获。又反复试了好多次，出错的表面原因多种多样。
4. 思考一下相关的理论知识。



重点放在`[BITS 32]`。

![image-20210225004914861](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210225004914861.png)

问题被解决了！解决方法就在上图中。编译器提供了`bits伪指令。`bits`指令之后的指令都将按照`bits`设定的模式编译。

那个耗费11个小时的错误，错在把不应该编译成32位的指令放在了`[bits 32]`后面，也就是把不该编译成32位指令的指令编译成了32位。这样具体造成了什么错误，我没有去考虑。不过，把需要编译成32位的指令放到了文件最后面、确保不会把其他指令误编译成32位，问题就消失了。

```assembly
[SECTION .s32]

ALIGN   32

[BITS   32]

LABEL_PM_START:
				jmp $
        jmp $
        jmp $
        jmp $
        
        mov ax, SelectFlatWR
        mov ds, ax
        mov es, ax
        mov fs, ax
        mov ss, ax
        mov ax, SelectVideo
        mov gs, ax

        mov gs, ax
        mov al, 'K'
        mov ah, 0Ah
        mov [gs:(80 * 19 + 25) * 2], ax

        jmp $
        jmp $
        jmp $
        jmp $
```



上面的代码正确执行之后，段寄存器被设置成了下面那样的。



```shell
268 00014033826i[BIOS  ] Booting from 0000:7c00
269 00300648000p[XGUI  ] >>PANIC<< POWER button turned off.
270 00300648000i[CPU0  ] CPU is in protected mode (active)
271 00300648000i[CPU0  ] CS.mode = 32 bit
272 00300648000i[CPU0  ] SS.mode = 32 bit
273 00300648000i[CPU0  ] EFER   = 0x00000000
274 00300648000i[CPU0  ] | EAX=60000a4b  EBX=00000600  ECX=00090002  EDX=0000000a
275 00300648000i[CPU0  ] | ESP=0000ffce  EBP=00000000  ESI=000e007c  EDI=0000007a
276 00300648000i[CPU0  ] | IOPL=0 id vip vif ac vm rf nt of df if tf sf zf af PF cf
277 00300648000i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
278 00300648000i[CPU0  ] |  CS:0008( 0001| 0|  0) 00000000 ffffffff 1 1
279 00300648000i[CPU0  ] |  DS:0010( 0002| 0|  0) 00000000 ffffffff 1 1
280 00300648000i[CPU0  ] |  SS:0010( 0002| 0|  0) 00000000 ffffffff 1 1
281 00300648000i[CPU0  ] |  ES:0010( 0002| 0|  0) 00000000 ffffffff 1 1
282 00300648000i[CPU0  ] |  FS:0010( 0002| 0|  0) 00000000 ffffffff 1 1
283 00300648000i[CPU0  ] |  GS:001b( 0003| 0|  3) 000b8000 0000ffff 0 0
284 00300648000i[CPU0  ] | EIP=0009037f (0009037f)
285 00300648000i[CPU0  ] | CR0=0x60000011 CR2=0x00000000
286 00300648000i[CPU0  ] | CR3=0x00000000 CR4=0x00000000
287 00300648000i[CMOS  ] Last time is 1614186688 (Wed Feb 24 09:11:28 2021)
288 00300648000i[XGUI  ] Exit
289 00300648000i[SIM   ] quit_sim called with exit code 1
```



```shell
00000000: EB 62 90 46 6F 72 72 65 73 74 59 00 02 01 01 00  .b.ForrestY.....
00000010: 02 E0 00 40 0B F0 09 00 12 00 02 00 00 00 00 00  ...@............
00000020: 00 00 00 00 00 00 29 00 00 00 00 4F 72 61 6E 67  ......)....Orang
00000030: 65 53 30 2E 30 32 46 41 54 31 32 20 20 20 00 00  eS0.02FAT12   ..
00000040: 00 00 00 00 00 00 FF FF 00 00 00 9A CF 00 FF FF  ................
00000050: 00 00 00 92 CF 00 FF FF 00 80 0B F2 00 00 1F 00  ................
00000060: 3E 01 09 00 B8 00 B8 8E E8 B4 0C B0 58 65 A3 28  >...........Xe.(
00000070: 0A B8 00 80 8E C0 B8 00 90 8E D8 B4 00 B2 00 CD  ................
00000080: 13 B8 13 00 B1 01 BB 00 00 E8 12 01 B9 04 00 BB  ................
00000090: 90 0B BF 00 00 83 F9 00 74 72 51 BE 49 02 B9 0B  ........trQ.I...
000000a0: 00 BA 00 00 AC 26 3A 05 75 0A 49 47 42 83 FA 0B  .....&:.u.IGB...
000000b0: 74 1C EB F0 B0 45 B4 0C 65 89 07 81 C3 A0 00 59  t....E..e......Y
000000c0: 83 F9 00 49 74 46 83 E7 E0 83 C7 20 EB C7 B0 53  ...ItF..... ...S
000000d0: B4 0A 65 A3 A6 0E 83 E7 E0 83 C7 1A 89 FE B8 00  ..e.............
000000e0: 80 1E 8E D8 AD 1F 50 BB 00 00 53 83 C0 13 83 C0  ......P...S.....
000000f0: 0E 83 E8 02 B1 01 5B E8 A4 00 81 C3 00 02 58 53  ......[.......XS
00000100: E8 51 00 5B 50 3D F8 0F 73 0C EB DE B0 4E B4 0A  .Q.[P=..s....N..
00000110: 65 A3 A8 0E EB 22 87 DB 0F 01 16 5E 01 FA E4 92  e....".....^....
00000120: 0C 02 E6 92 0F 20 C0 66 83 C8 01 0F 22 C0 87 DB  ..... .f...."...
00000130: 66 EA 60 03 09 00 08 00 EB FE 48 65 6C 6C 6F 2C  f.`.......Hello,
00000140: 57 6F 72 6C 64 20 4F 53 21 4B 45 52 4E 45 4C 20  World OS!KERNEL
00000150: 20 42 49 4E 50 B4 00 B2 00 CD 13 58 BA 00 00 BB   BINP......X....
00000160: 03 00 F7 E3 BB 02 00 F7 F3 89 16 00 00 BA 00 00  ................
00000170: B9 00 02 F7 F1 83 C0 01 B1 02 BB 00 00 06 52 50  ..............RP
00000180: B8 00 10 8E C0 58 E8 15 00 5A 01 D3 26 8B 07 07  .....X...Z..&...
00000190: 80 3E 00 00 00 74 03 C1 E8 04 25 FF 0F C3 50 55  .>...t....%...PU
000001a0: 53 89 E5 83 EC 02 88 4E FE B3 12 F6 F3 88 C5 D0  S......N........
000001b0: ED 88 C6 80 E6 01 B2 00 FE C4 88 E1 8A 46 FE 83  .............F..
000001c0: C4 02 B4 02 5B CD 13 5D 58 C3 55 89 E5 50 51 56  ....[..]X.U..PQV
000001d0: 57 8B 7E 04 8B 76 06 8B 4E 08 06 8E C7 BF 00 00  W.~..v..N.......
000001e0: 3E 8A 04 26 88 05 46 47 49 83 F9 00 74 02 EB F0  >..&..FGI...t...
000001f0: 07 8B 46 04 5F 5E 59 58 5D C3 50 51 56 B8 00 80  ..F._^YX].PQV...
00000200: 1E 8E D8 87 DB 8B 0E 2C 00 31 F6 8B 36 1C 00 83  .......,.1..6...
00000210: C6 00 87 DB 8B 44 10 50 B8 00 00 03 44 04 50 8B  .....D.P....D.P.
00000220: 44 08 50 87 DB E8 A2 FF 87 DB 83 C4 06 49 83 F9  D.P..........I..
00000230: 00 74 05 83 C6 20 EB DC 1F 5E 59 58 C3 B5 00 B1  .t... ...^YX....
00000240: 02 B6 01 B2 00 B0 01 B4 02 BB 00 80 CD 13 C3 00  ................
00000250: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000260: 66 B8 10 00 8E D8 8E C0 8E E0 8E D0 66 B8 1B 00  f...........f...
00000270: 8E E8 8E E8 B0 4B B4 0A 65 66 A3 12 0C 00 00 EB  .....K..ef......
00000280: FE EB FE EB FE EB FE                             .......
```



```assembly
; filename: bits-demo.asm
; nasm -o bits-demo bits-demo.asm
mov ax, 3
mov cx, 1
jmp $
jmp $

SelectFlatWR    equ     $ - $$
SelectVideo     equ     $ - $$

[SECTION .s32]

ALIGN   32

[BITS   32]

LABEL_PM_START:
                                jmp $
        jmp $
        jmp $
        jmp $

        mov ax, SelectFlatWR
        mov ds, ax
        mov es, ax
        mov fs, ax
        mov ss, ax
        mov ax, SelectVideo
        mov gs, ax

        mov gs, ax
        mov al, 'K'
        mov ah, 0Ah
        mov [gs:(80 * 19 + 25) * 2], ax

        jmp $  
```

用xxd查看二进制数据

```shell
[root@localhost v3]# xxd -u -a -g 1 -c 16 bits-demo
00000000: B8 03 00 B9 01 00 EB FE EB FE 00 00 00 00 00 00  ................
00000010: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000020: EB FE EB FE EB FE EB FE 66 B8 0A 00 8E D8 8E C0  ........f.......
00000030: 8E E0 8E D0 66 B8 0A 00 8E E8 8E E8 B0 4B B4 0A  ....f........K..
00000040: 65 66 A3 12 0C 00 00 EB FE EB FE EB FE EB FE     ef.............
```

进入保护模式后，CPU在32位模式下，有66前缀的，表示这些机器指令码应该解释为16位的。没有66前缀的，就应该解释为32位的。

```assembly
; filename: bits-demo-16.asm
; nasm -o bits-demo-16 bits-demo-16.asm
mov ax, 3
mov cx, 1
jmp $
jmp $

SelectFlatWR    equ     $ - $$
SelectVideo     equ     $ - $$

[SECTION .s32]

;ALIGN   32

;[BITS   32]

LABEL_PM_START:
                                jmp $
        jmp $
        jmp $
        jmp $

        mov ax, SelectFlatWR
        mov ds, ax
        mov es, ax
        mov fs, ax
        mov ss, ax
        mov ax, SelectVideo
        mov gs, ax

        mov gs, ax
        mov al, 'K'
        mov ah, 0Ah
        mov [gs:(80 * 19 + 25) * 2], ax

        jmp $  
```



查看二进制数据：

```shell
[root@localhost v3]# xxd -u -a -g 1 -c 16 bits-demo-16
00000000: B8 03 00 B9 01 00 EB FE EB FE 00 00 EB FE EB FE  ................
00000010: EB FE EB FE B8 0A 00 8E D8 8E C0 8E E0 8E D0 B8  ................
00000020: 0A 00 8E E8 8E E8 B0 4B B4 0A 65 A3 12 0C EB FE  .......K..e.....
```



上面的三份代码，第一份是loader加载器中截取出来的，二进制数据是loader的。后面两份是把第一份代码提取出来后单独编译，二进制数据是单独编译的结果。共同点是：有`x66`前缀；多个`jmp $`相同。具体数据有差异。



太不容易了，运行效果，截个图：

![image-20210225012115385](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210225012115385.png)



### v4-loader

这个版本实现：

在进入保护模式后，

1. 重新放置内核
2. 执行内核

复制v3的代码，修改为32位。因为`Memcpy`等代码，已经写过16位版本，实在要写32位版本，没有太多收益。复制过来修改，能加快进度。

首先，把要修改的代码移动到32位区域，然后修改代码。

42分钟就修改好了代码，就是修改`Memcpy`和`InitKernel`。可是，不知道怎么验证是否正确。

进入保护模式后，`xchg bx, bx`好像不能用了。

怎么验证？

1. 查看a.img的二进制数据是否正确。
2. 运行效果。



运行效果不正确。这种方式是个黑盒子。

查看二进制数据，不知道怎么查看。

方法是，对应位置的数据是不是期望的数据。

重新放置内核，是把ELF文件的程序段放置到内存位置为`0x30000`的位置。这是个虚拟内存地址。在bochs中查看的方法是：在保护模式前断点，然后单步执行。太耗费时间了，因为重新放置内核只能单步执行。再确认一下再保护模式中能否断点。

用xxd查看的方法是：哪个段？反正不是视频段，基地址是0x0，那么，`0x30000`计算成`0x30000`。

上面的方法有误，在软盘中是看不到在内存中重新放置后的内核的。

只能再看看”在保护模式下怎么断点调试“。

在保护模式下能断点。我没有找到验证代码是否正确的逻辑。

需想好思路，再断点调试。

验证方法：

1. 看三个参数有没有问题。

经过实例分析后，重新放置内核没有问题。但是跳转到内核时，出现错误：

```shell
<bochs:27> s
00014812081e[CPU0  ] load_seg_reg(DS, 0x007c): segment not present
00014812081e[CPU0  ] fetch_raw_descriptor: GDT: index (f007) 1e00 > limit (1f)
00014812081e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x08)
00014812081i[CPU0  ] CPU is in protected mode (active)
00014812081i[CPU0  ] CS.mode = 32 bit
00014812081i[CPU0  ] SS.mode = 32 bit
00014812081i[CPU0  ] EFER   = 0x00000000
00014812081i[CPU0  ] | EAX=00030000  EBX=00000600  ECX=00000000  EDX=0000000a
00014812081i[CPU0  ] | ESP=0000ffbe  EBP=00000000  ESI=00080034  EDI=0000007a
00014812081i[CPU0  ] | IOPL=0 id vip vif ac vm RF nt of df if tf sf ZF af PF cf
00014812081i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
00014812081i[CPU0  ] |  CS:0008( 0001| 0|  0) 00000000 ffffffff 1 1
00014812081i[CPU0  ] |  DS:0010( 0002| 0|  0) 00000000 ffffffff 1 1
00014812081i[CPU0  ] |  SS:0010( 0002| 0|  0) 00000000 ffffffff 1 1
00014812081i[CPU0  ] |  ES:0010( 0002| 0|  0) 00000000 ffffffff 1 1
00014812081i[CPU0  ] |  FS:0010( 0002| 0|  0) 00000000 ffffffff 1 1
00014812081i[CPU0  ] |  GS:001b( 0003| 0|  3) 000b8000 0000ffff 0 0
00014812081i[CPU0  ] | EIP=0009036f (0009036f)
00014812081i[CPU0  ] | CR0=0x60000011 CR2=0x00000000
00014812081i[CPU0  ] | CR3=0x00000000 CR4=0x00000000
(0).[14812081] [0x00000009036f] 0008:000000000009036f (unk. ctxt): pop ds                    ; 1f
00014812081e[CPU0  ] exception(): 3rd (13) exception with no resolution, shutdown status is 00h, resetting
00014812081i[SYS   ] bx_pc_system_c::Reset(HARDWARE) called
00014812081i[CPU0  ] cpu hardware reset
```



先不管上面的错误。先找到执行内核指令的方法。

无非就是用`jmp`指令。段选择子用cs吗？即`jmp SelectFlatX:0x30400`。是的。

上面的错误，是由于在已经设置gs后，再次用非选择子的值设置gs。GDT的最大索引是`1f`。

##### 实例分析

###### a.img

```shell
[root@localhost v4]# xxd -u -a -g 1 -c 16 a.img
00000000: EB 3C 90 46 6F 72 72 65 73 74 59 00 02 01 01 00  .<.ForrestY.....
00000010: 02 E0 00 40 0B F0 09 00 12 00 02 00 00 00 00 00  ...@............
00000020: 00 00 00 00 00 00 29 00 00 00 00 4F 72 61 6E 67  ......)....Orang
00000030: 65 53 30 2E 30 32 46 41 54 31 32 20 20 20 B8 00  eS0.02FAT12   ..
00000040: B8 8E E8 B8 00 90 8E C0 B4 00 B2 00 CD 13 B8 13  ................
00000050: 00 B1 01 BB 00 01 E8 07 01 B9 03 00 BF 00 01 83  ................
00000060: F9 00 74 72 51 BE 0B 7D B9 0B 00 BA 00 00 BB 90  ..trQ..}........
00000070: 0B AC 26 3A 05 75 0A 49 47 42 83 FA 0B 74 19 EB  ..&:.u.IGB...t..
00000080: F0 B0 44 B4 0A 65 A3 50 0F 59 83 F9 00 49 74 46  ..D..e.P.Y...ItF
00000090: 83 E7 E0 83 C7 20 EB C7 B0 53 B4 0A 65 A3 46 0F  ..... ...S..e.F.
000000a0: 83 E7 E0 83 C7 1A 89 FE B8 00 90 1E 8E D8 AD 1F  ................
000000b0: 50 BB 00 01 53 83 C0 13 83 C0 0E 83 E8 02 B1 01  P...S...........
000000c0: 5B E8 9C 00 81 C3 00 02 58 53 E8 49 00 5B 50 3D  [.......XS.I.[P=
000000d0: F8 0F 73 0C EB DE B0 4E B4 0A 65 A3 48 0F EB 1A  ..s....N..e.H...
000000e0: 83 C0 13 83 C0 0E 83 E8 02 B1 01 B0 4F B4 0A 65  ............O..e
000000f0: A3 42 0F EA 00 01 00 90 EB 00 EB FE 48 65 6C 6C  .B..........Hell
00000100: 6F 2C 57 6F 72 6C 64 20 4F 53 21 4C 4F 41 44 45  o,World OS!LOADE
00000110: 52 20 20 42 49 4E 50 B4 00 B2 00 CD 13 58 BA 00  R  BINP......X..
00000120: 00 BB 03 00 F7 E3 BB 02 00 F7 F3 89 16 00 00 BA  ................
00000130: 00 00 B9 00 02 F7 F1 83 C0 01 B1 02 BB 00 00 06  ................
00000140: 52 50 B8 00 10 8E C0 58 E8 15 00 5A 01 D3 26 8B  RP.....X...Z..&.
00000150: 07 07 80 3E 00 00 00 74 03 C1 E8 04 25 FF 0F C3  ...>...t....%...
00000160: 50 55 53 89 E5 66 83 EC 02 88 4E FE B3 12 F6 F3  PUS..f....N.....
00000170: 88 C5 D0 ED 88 C6 80 E6 01 B2 00 FE C4 88 E1 8A  ................
00000180: 46 FE 66 83 C4 02 B4 02 5B CD 13 5D 58 C3 B5 00  F.f.....[..]X...
00000190: B1 02 B6 01 B2 00 B0 01 B4 02 BB 00 90 CD 13 C3  ................
000001a0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
*
000001f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 55 AA  ..............U.
00000200: 00 00 00 00 40 00 FF 6F 00 07 F0 FF 00 00 00 00  ....@..o........
00000210: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
*
00001400: 00 00 00 00 40 00 FF 6F 00 07 F0 FF 00 00 00 00  ....@..o........
00001410: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
*
00002600: 41 6C 00 6F 00 61 00 64 00 65 00 0F 00 AB 72 00  Al.o.a.d.e....r.
00002610: 2E 00 62 00 69 00 6E 00 00 00 00 00 FF FF FF FF  ..b.i.n.........
00002620: 4C 4F 41 44 45 52 20 20 42 49 4E 20 00 00 FC 15  LOADER  BIN ....
00002630: 59 52 59 52 00 00 FC 15 59 52 03 00 9F 02 00 00  YRYR....YR......
00002640: 41 6B 00 65 00 72 00 6E 00 65 00 0F 00 DA 6C 00  Ak.e.r.n.e....l.
00002650: 2E 00 62 00 69 00 6E 00 00 00 00 00 FF FF FF FF  ..b.i.n.........
00002660: 4B 45 52 4E 45 4C 20 20 42 49 4E 20 00 00 FC 15  KERNEL  BIN ....
00002670: 59 52 59 52 00 00 FC 15 59 52 05 00 A8 04 00 00  YRYR....YR......
00002680: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
*
00004400: EB 62 90 46 6F 72 72 65 73 74 59 00 02 01 01 00  .b.ForrestY.....
00004410: 02 E0 00 40 0B F0 09 00 12 00 02 00 00 00 00 00  ...@............
00004420: 00 00 00 00 00 00 29 00 00 00 00 4F 72 61 6E 67  ......)....Orang
00004430: 65 53 30 2E 30 32 46 41 54 31 32 20 20 20 00 00  eS0.02FAT12   ..
00004440: 00 00 00 00 00 00 FF FF 00 00 00 9A CF 00 FF FF  ................
00004450: 00 00 00 92 CF 00 FF FF 00 80 0B F2 00 00 1F 00  ................
00004460: 3E 01 09 00 B8 00 B8 8E E8 B4 0C B0 58 65 A3 28  >...........Xe.(
00004470: 0A B8 00 80 8E C0 B8 00 90 8E D8 B4 00 B2 00 CD  ................
00004480: 13 B8 13 00 B1 01 BB 00 00 E8 12 01 B9 04 00 BB  ................
00004490: 90 0B BF 00 00 83 F9 00 74 72 51 BE 49 02 B9 0B  ........trQ.I...
000044a0: 00 BA 00 00 AC 26 3A 05 75 0A 49 47 42 83 FA 0B  .....&:.u.IGB...
000044b0: 74 1C EB F0 B0 45 B4 0C 65 89 07 81 C3 A0 00 59  t....E..e......Y
000044c0: 83 F9 00 49 74 46 83 E7 E0 83 C7 20 EB C7 B0 53  ...ItF..... ...S
000044d0: B4 0A 65 A3 A6 0E 83 E7 E0 83 C7 1A 89 FE B8 00  ..e.............
000044e0: 80 1E 8E D8 AD 1F 50 BB 00 00 53 83 C0 13 83 C0  ......P...S.....
000044f0: 0E 83 E8 02 B1 01 5B E8 A4 00 81 C3 00 02 58 53  ......[.......XS
00004500: E8 51 00 5B 50 3D F8 0F 73 0C EB DE B0 4E B4 0A  .Q.[P=..s....N..
00004510: 65 A3 A8 0E EB 22 87 DB 0F 01 16 5E 01 FA E4 92  e....".....^....
00004520: 0C 02 E6 92 0F 20 C0 66 83 C8 01 0F 22 C0 87 DB  ..... .f...."...
00004530: 66 EA E0 02 09 00 08 00 EB FE 48 65 6C 6C 6F 2C  f.........Hello,
00004540: 57 6F 72 6C 64 20 4F 53 21 4B 45 52 4E 45 4C 20  World OS!KERNEL
00004550: 20 42 49 4E 50 B4 00 B2 00 CD 13 58 BA 00 00 BB   BINP......X....
00004560: 03 00 F7 E3 BB 02 00 F7 F3 89 16 00 00 BA 00 00  ................
00004570: B9 00 02 F7 F1 83 C0 01 B1 02 BB 00 00 06 52 50  ..............RP
00004580: B8 00 10 8E C0 58 E8 15 00 5A 01 D3 26 8B 07 07  .....X...Z..&...
00004590: 80 3E 00 00 00 74 03 C1 E8 04 25 FF 0F C3 50 55  .>...t....%...PU
000045a0: 53 89 E5 83 EC 02 88 4E FE B3 12 F6 F3 88 C5 D0  S......N........
000045b0: ED 88 C6 80 E6 01 B2 00 FE C4 88 E1 8A 46 FE 83  .............F..
000045c0: C4 02 B4 02 5B CD 13 5D 58 C3 B5 00 B1 02 B6 01  ....[..]X.......
000045d0: B2 00 B0 01 B4 02 BB 00 80 CD 13 C3 00 00 00 00  ................
000045e0: 66 B8 10 00 8E D8 8E C0 8E E0 8E D0 66 B8 1B 00  f...........f...
000045f0: 8E E8 8E E8 B0 4B B4 0A 65 66 A3 12 0C 00 00 66  .....K..ef.....f
00004600: 87 DB E8 20 00 00 00 66 87 DB 8E E8 B0 47 B4 0A  ... ...f.....G..
00004610: 65 66 A3 08 0C 00 00 66 87 DB E9 E1 00 03 00 EB  ef.....f........
00004620: FE EB FE EB FE EB FE 50 51 56 66 87 DB 66 8B 0D  .......PQVf..f..
00004630: 2C 00 08 00 0F B7 C9 31 F6 8B 35 1C 00 08 00 81  ,......1..5.....
00004640: C6 00 00 08 00 66 87 DB 8B 46 10 50 B8 00 00 08  .....f...F.P....
00004650: 00 03 46 04 50 8B 46 08 50 E8 16 00 00 00 83 C4  ..F.P.F.P.......
00004660: 0C 49 83 F9 00 74 05 83 C6 20 EB DC 66 87 DB 1F  .I...t... ..f...
00004670: 5E 59 58 C3 55 89 E5 50 51 56 57 8B 7D 08 8B 75  ^YX.U..PQVW.}..u
00004680: 0C 8B 4D 10 06 3E 8A 06 26 88 07 46 47 49 83 F9  ..M..>..&..FGI..
00004690: 00 74 02 EB F0 07 8B 45 08 5F 5E 59 58 5D C3 00  .t.....E._^YX]..
000046a0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
*
00004800: 7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
00004810: 02 00 03 00 01 00 00 00 00 04 03 00 34 00 00 00  ............4...
00004820: 30 04 00 00 00 00 00 00 34 00 20 00 01 00 28 00  0.......4. ...(.
00004830: 03 00 02 00 01 00 00 00 00 00 00 00 00 00 03 00  ................
00004840: 00 00 03 00 1D 04 00 00 1D 04 00 00 05 00 00 00  ................
00004850: 00 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00004860: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
*
00004c00: B4 0F B0 43 65 66 A3 D0 0C 00 00 65 66 A3 70 0D  ...Cef.....ef.p.
00004c10: 00 00 65 66 A3 60 0E 00 00 EB FE EB FE 00 2E 73  ..ef.`.........s
00004c20: 68 73 74 72 74 61 62 00 2E 74 65 78 74 00 00 00  hstrtab..text...
00004c30: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00004c40: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00004c50: 00 00 00 00 00 00 00 00 0B 00 00 00 01 00 00 00  ................
00004c60: 06 00 00 00 00 04 03 00 00 04 00 00 1D 00 00 00  ................
00004c70: 00 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00  ................
00004c80: 01 00 00 00 03 00 00 00 00 00 00 00 00 00 00 00  ................
00004c90: 1D 04 00 00 11 00 00 00 00 00 00 00 00 00 00 00  ................
00004ca0: 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00004cb0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
*
00167ff0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
```



elf相关：

```shell
00004800: 7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
00004810: 02 00 03 00 01 00 00 00 00 04 03 00 34 00 00 00  ............4...
00004820: 30 04 00 00 00 00 00 00 34 00 20 00 01 00 28 00  0.......4. ...(.
00004830: 03 00 02 00 01 00 00 00 00 00 00 00 00 00 03 00  ................
00004840: 00 00 03 00 1D 04 00 00 1D 04 00 00 05 00 00 00  ................
00004850: 00 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00004860: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
*
00004c00: B4 0F B0 43 65 66 A3 D0 0C 00 00 65 66 A3 70 0D  ...Cef.....ef.p.
00004c10: 00 00 65 66 A3 60 0E 00 00 EB FE EB FE 00 2E 73  ..ef.`.........s
00004c20: 68 73 74 72 74 61 62 00 2E 74 65 78 74 00 00 00  hstrtab..text...
00004c30: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00004c40: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00004c50: 00 00 00 00 00 00 00 00 0B 00 00 00 01 00 00 00  ................
00004c60: 06 00 00 00 00 04 03 00 00 04 00 00 1D 00 00 00  ................
00004c70: 00 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00  ................
00004c80: 01 00 00 00 03 00 00 00 00 00 00 00 00 00 00 00  ................
00004c90: 1D 04 00 00 11 00 00 00 00 00 00 00 00 00 00 00  ................
00004ca0: 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00004cb0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
*
00167ff0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
```

1. 程序段的个数，在ELF文件的偏移量是2Ch个，

   1. ```shell
      00004800: 7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
      00004810: 02 00 03 00 01 00 00 00 00 04 03 00 34 00 00 00  ............4...
      00004820: 30 04 00 00 00 00 00 00 34 00 20 00 01 00 28 00  0.......4. ...(.
      ```

   2. 第3行的第ch个，两个字节，1个字，`0001`。

2. 第一个程序头，在ELF文件的偏移量是34h个字节。

   1. ```
      00004830: -- -- -- -- 01 00 00 00 00 00 00 00 00 00 03 00  ................
      00004840: 00 00 03 00 1D 04 00 00 1D 04 00 00 05 00 00 00  ................
      00004850: 00 10 00 00
      ```

   2. 程序段的长度，在程序头中的偏移量是10h个字节，4个字节，`00 00 04 1D`。

   3. 程序段的偏移量，在文件中的偏移量，在程序头中的偏移量是4h个字节，`00 00 00 00`。

   4. 程序段在内存中的虚拟地址，在程序头中的偏移量是8h个字节，`00 03 00 00`。

   5. 二进制代码是小端法，低位在左边，高位在右边。

3. 重点分析程序段的长度。`041Dh`。起始行是`0004800`，加上`041Dh`个字节。

   1. 041Dh = 0410h + Dh = 0410h + Dh。
   2. elf文件的`0x0`位置，也就是第一个字节的位置（就是行号） + 0410h ，0410h + 0004800h = 0004C10h。
   3. 重新放置内核的时候，elf文件的`0x0`位置必须是它在内存中的偏移量，基地址是elf文件被加载到内存的物理地址。物理地址的计算方法是"段选择子：偏移量"。其中，偏移量是读取内核文件到内存时的物理地址。
   4. 这些东西，总感觉理解得不透彻。要我马上讲解这些，时间久了，可能讲不清楚。

4. 多次执行make编译、拷贝文件到a.img可能导致a.img中由重复的内核文件。请注意。

数据全部弄清楚了，只需对照断点调试看到的数据和xxd查看的数据。

```shell
xp /1wx 0x0010:0x30000
xp /1wx 0x0010:0x30400
```

### v5-loader

这个版本实现：

1. 内核使用C语言和汇编混合编程。暂时无法完成。
2. 把立即数用变量名替换

#### C和汇编混合

怎么混合？

bar.c是C语言写的源码文件，foo.asm是汇编写的源码文件。在foo.asm中使用bar.c中创建的函数，在bar.c中使用foo.asm提供的函数。

foo.asm创建的函数用`global`导出，使用bar.c中创建的函数前使用`extern`导入。

```assembly
extern choose

[section .data]

GreaterNumber	equ	51
SmallerNumber equ	23

[section .text]

global _start
global _displayStr

_start:
	push	GreaterNumber
	push	SmallerNumber
	call choose
	add [esp+8]	; 人工清除参数占用的栈空间
	
	; 必须调用 exit，否则会出现错提示，程序能运行。
	mov eax, 1
	mov ebx, 0
	int 0x80
	
	ret
	
; _displayStr(char *str, int len)	
_displayStr:
	mov eax, 4
	mov ebx, 1
	; 按照C函数调用规则，最后一个参数最先入栈，它在栈中的地址最大。
	mov ecx, [ebp + 4]		; str
	mov edx, [ebp + 8]		; len。ebp + 0 是 cs:ip中的ip
	int 0x80
	
	ret	; 一定不能少
```



```c
void choose(int a, int b)
{
  if(a > b){
    // 哪些函数能用，哪些函数不能用，不清楚。用C语言写代码，仍受束缚啊。
    // 比如，我想把a和字符串混合起来。在PHP中直接用点号就能连接起来。
    _displayStr("first", 5);
  }else{
    _displayStr("second", 6);
  }
  
  return;
}
```



怎么编译？

```shell
nasm -f elf foo.o foo.asm
gcc -o bar.o bar.c -m32
ld -s -o kernel.bin foo.o bar.o -m elf_i386
```



上面的代码经过编译后，能运行。但是作为操作系统内核，不能运行。





要在目前的内核中运行用使用了系统调用的代码，似乎不行。

```assembly
[section .data]
Str:    db      "Hello,World"
Len     equ     $ - Str

[section .text]

global _start

_start:
        mov eax, 4
        mov ebx, 1
        mov ecx, Str
        mov edx, Len
        int 0x80

        mov eax, 1
        mov ebx, 0
        int 0x80

        ret
```



单独编译后能运行，作为操作系统内核，不能运行，出现下面的报错信息：

```shell
268 00014033826i[BIOS  ] Booting from 0000:7c00
269 00014812264e[CPU0  ] interrupt(): vector must be within IDT table limits, IDT.limit = 0x3ff
270 00014812264e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x0d)
271 00014812264e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x08)
272 00014812264i[CPU0  ] CPU is in protected mode (active)
273 00014812264i[CPU0  ] CS.mode = 32 bit
274 00014812264i[CPU0  ] SS.mode = 32 bit
275 00014812264i[CPU0  ] EFER   = 0x00000000
276 00014812264i[CPU0  ] | EAX=00000004  EBX=00000001  ECX=00031424  EDX=0000000b
277 00014812264i[CPU0  ] | ESP=0000ffce  EBP=00000000  ESI=000e007c  EDI=0000007a
278 00014812264i[CPU0  ] | IOPL=0 id vip vif ac vm RF nt of df if tf sf ZF af PF cf
279 00014812264i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
280 00014812264i[CPU0  ] |  CS:0008( 0001| 0|  0) 00000000 ffffffff 1 1
281 00014812264i[CPU0  ] |  DS:0010( 0002| 0|  0) 00000000 ffffffff 1 1
282 00014812264i[CPU0  ] |  SS:0010( 0002| 0|  0) 00000000 ffffffff 1 1
283 00014812264i[CPU0  ] |  ES:0010( 0002| 0|  0) 00000000 ffffffff 1 1
284 00014812264i[CPU0  ] |  FS:0010( 0002| 0|  0) 00000000 ffffffff 1 1
285 00014812264i[CPU0  ] |  GS:001b( 0003| 0|  3) 000b8000 0000ffff 0 0
```

试图用C和汇编混合编写内核，无业务逻辑，纯语法，就浪费了2个小时。调试仍然低效。似乎不能使用`xchg bx, bx`断点。

再次强调，遇到错误，不可无脑调试。昨天花了11个小时都没能解决问题。我没有那么多时间被浪费！

#### 切换堆栈和GDT

进入保护模式后，GDT放在loader中。想在kernel中使用GDT，可以吗？在不同文件中（例如boot、loader、kernel)，复制到内存后，在不同的内存中片中，一个内存片中要使用另一个内存片中的变量，不行。即使，在不同文件中的变量名相同，这些相同的变量名到了内存中，内存地址不同。

上面的解释，比较混乱。

在loader中，能获取GdtPtr的内存地址（段选择子：偏移量）。

仍然比较混乱。

总之，我不明白，在内核中为啥需要重新获取GDT。

能够在内核中重新获取GDT，但是比较繁琐，需要计算。切换GDT，就是只计算一次、把计算结果保存起来。对，就这么理解。

从GDT的基地址P开始，X个字节的数据，都是GDT。所以，复制GDT的方法是：从P开始，把X个字节复制到内存地址N开始的内存。X是多少？它就是lgdt寄存器中的数据的低16位。X不是32位吗？X是8的整数倍。这个理解似乎无用。这么想吧。GDT中的元素一共有X/8个。GDT中每个相邻的元素的内存地址的差值是8（单位是字节），相差8个字节，是64位。

GDT的长度是X，最后一个描述符在GDT中的索引是X/8 - 1，GdtPtr的界限是指GDT的长度-1。

所谓界限，就是GDT的长度减去1；GDT的长度就是界限 + 1，因为，要把初始点也包括在内（不对，因为初始点的值被设置为0），那么，索引1是第2个，索引2是第3个。不能用索引。这种细节理解起来，我感觉比较麻烦和纠结。一共有X个字节，第1个字节是第0个，第X个字节是多少个？第X-1个。用举例法就能知道这个结论。第1个字节是第0个，第2个字节是第1个，第3个字节是第2个。

为啥只用16位来表示GDT的长度？16位能表示多少个描述符？16位能表示的最大数是2的16次方-1。段界限的最大值是2的16次方-1。

段界限的最大值 = 2的16次方 - 1。GDT的长度 = 段界限的最大值 + 1。GDT的长度 = 2的16次方。

2的16次方字节/8 = 2的13次方 = 8192。操作系统最多能有8192个字节。为啥？规定吧。8192个字节，够用了吧。

理解这么简单的东西，花了大概40分钟。初衷不是为了理解这个东西。

复制GDT，从GDT的开始位置，复制GDT的长度个字节，这就是复制GDT。

GdtPtr的基地址和界限，很容易获取。然而，获取它们之后，为啥又要重新给二者赋值呢？

```c
u16* p_gdt_limit = (u16*)(&gdt_ptr[0]);
u32* p_gdt_base  = (u32*)(&gdt_ptr[2]);
*p_gdt_limit = GDT_SIZE * sizeof(DESCRIPTOR) - 1;
*p_gdt_base  = (u32)&gdt;
```

重复赋值，是因为二者的内容不同吗？理解不了这里，是C语言基础差？没有其他办法，断点调试观察数据吧。

对了，逻辑，我清楚了吧。把GDT的相关数据复制到新的GDT系列变量中。我可以按自己的方法实现这个目的，不一定非要去理解于上神的代码。

##### 理解代码

```c
PUBLIC	void*	memcpy(void* pDst, void* pSrc, int iSize);

PUBLIC	void	disp_str(char * pszInfo);

PUBLIC	u8		gdt_ptr[6];	/* 0~15:Limit  16~47:Base */
PUBLIC	DESCRIPTOR	gdt[GDT_SIZE];

PUBLIC void cstart()
{

	disp_str("\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n"
		 "-----\"cstart\" begins-----\n");

	/* 将 LOADER 中的 GDT 复制到新的 GDT 中 */
	memcpy(&gdt,				   /* New GDT */
	       (void*)(*((u32*)(&gdt_ptr[2]))),    /* Base  of Old GDT */
	       *((u16*)(&gdt_ptr[0])) + 1	   /* Limit of Old GDT */
		);
	/* gdt_ptr[6] 共 6 个字节：0~15:Limit  16~47:Base。用作 sgdt/lgdt 的参数。*/
	u16* p_gdt_limit = (u16*)(&gdt_ptr[0]);
	u32* p_gdt_base  = (u32*)(&gdt_ptr[2]);
	*p_gdt_limit = GDT_SIZE * sizeof(DESCRIPTOR) - 1;
	*p_gdt_base  = (u32)&gdt;
}
```

`(void*)(*((u32*)(&gdt_ptr[2])))` 怎么理解？

`(u32*)(&gdt_ptr[2])`构造一个`u32*`类型数据，`u32*`是强制进行数据类型转换。`*(u32*)(&gdt_ptr[2])`是获取一个`u32*`类型变量的值即`gdt_ptr[2]`的地址即`&gdt_ptr[2]`。`(void*)`是函数`PUBLIC	void*	memcpy(void* pDst, void* pSrc, int iSize);`对参数的数据类型要求。

`*((u16*)(&gdt_ptr[0]))`等价于`(int*)*((u16*)(&gdt_ptr[0]))`。对它的理解与上面相同。经过测试，`int*`可有可无。

这是从语法层面理解。从业务层面呢？

1. `gdt_ptr`存储的GDT的物理位置，低16位是GDT的字节长度-1，高32位是GDT的物理地址（第一个元素的物理地址)。

2. `&gdt_ptr[0]`

   1. `&gdt_ptr[0]`是`gdt_ptr`的物理位置。gdt_ptr是有6个char元素的数组。它的内存地址(物理位置)是第一个元素内存地址。

   2. `(u16 *)(&gdt_ptr[0])`创建一个指针变量，这个指针变量的值是`&gdt_ptr[0]`。把这个指针变量的值赋值给`u16* p_gdt_limit`。

   3. `u16* p_gdt_limit = (u16*)(&gdt_ptr[0]);`的语法规则，可以用这段代码来理解：

      ```c
      int b = 7;
      int *a = (int *)(&b);
      ```

3. 重复赋值。

   ```c
   u16* p_gdt_limit = (u16*)(&gdt_ptr[0]);
   u32* p_gdt_base  = (u32*)(&gdt_ptr[2]);
   ```

   上面的代码已经对`p_gdt_limit`和`p_gdt_base`赋值了，下面的代码，我认为是重复的。

   ```c
   *p_gdt_limit = GDT_SIZE * sizeof(DESCRIPTOR) - 1;
   *p_gdt_base  = (u32)&gdt;
   ```

   `u32* p_gdt_base`存储GDT的内存地址，对指针赋值的语法是`*pointer = value`，所以，`*p_gdt_base  = (u32)&gdt;`。

   `u16* p_gdt_limit`存储的不是内存地址，而是GDT的长度-1，是一个数据。

##### 小结

花了4个小时理解这一小段代码。

1. 理解GDT复制的业务逻辑。
2. 这段C语言使用了复杂的指针，理解这些语法。
3. 不理解代码中的重复赋值。我想验证它们是不是真的重复赋值了，所以，我使用bochs和GDB断点调试C代码。这又耗费了很多时间。这个时间本不应该浪费的，因为我以前使用过GDB调试。因没做好笔记或不知道笔记在哪里，只能重新摸索。
4. 不理解为何到了内核要切换堆栈。直到现在，我仍然不理解。
5. 理解本来以为没有疑问的细节，GdtPtr结构的低16位为啥设置为16位。
6. 凌晨花3个多小时看过的“如何学习”的书，一点作用都没有发挥。笔记一定要记录好。

##### 资料

# SeaBIOS实现简单分析

SeaBIOS是一个16bit的x86 BIOS的开源实现，常用于QEMU等仿真器中使用。本文将结合[SeaBIOS Execution and code flow](https://www.seabios.org/Execution_and_code_flow)和[SeaBIOS的源码](https://github.com/coreboot/seabios/)对SeaBIOS的全过程进行简单分析。需要注意，本文不是深入的分析，对于一些比较复杂和繁琐的部分直接跳过了。

https://doc.coreboot.org/

OS 开发论坛：

https://forum.osdev.org/viewforum.php?f=1&sid=65ed27036930773c125a8dd8eab714e4



## 32位下段寄存器的组成图：（不考虑64位）

![img](https://img2020.cnblogs.com/blog/1424296/202004/1424296-20200423111012366-212964709.png)

分为4个部分

- Selector  16位/可见
- Attribute 16位/不可见
- Limit  32位/不可见
- Base 32位/不可见

## 实模式寻址

### 8086实模式寻址时：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
mov cx,0x2000
mov ds,cx
mov [0xc0],al
;以上，将段寄存器ds的值设置为0x2000，
;然后向该段内偏移地址为0x00c0的地址写入al的值，
;写入时，将ds的值左移4位，再加上0x00c0，即：0x200c0
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

### 32位实模式寻址不一样：

```
mov ax,cs
mov ds,ax
;第一句 ax=cs.Selector
;第二句 先 ds.Selector=ax
;然后 ds.Base=ds.Selector<<4，Base只用到了低20位。
```

32位环境下，当用汇编读取某一地址时：

```
mov dword ptr ds:[0x123456],eax
;真正读写的地址是ds.base+0x123456
```

##  32位保护模式下寻址

6个段寄存器ES CS DS SS FS GS叫做段选择器，和实模式不同，保护模式的内存访问有自己的方式，

在保护模式下，尽管访问内存时也需要指定一个段，但传送到段选择器的内容不是逻辑段地址，而是段选择子。

段选择子由三部分组成：

![img](https://img2020.cnblogs.com/blog/1424296/202004/1424296-20200423130119143-1514219737.png)

- 描述符索引——用来在描述符表中选择一个段描述符。
- 描述符表指示器——TI=0时，在GDT表中；TI=1时，在LDT表中。
- 请求特权级别——4个级别，0，1，2，3。

例：

```
mov cx,00000000000_10_000B         ;加载数据段选择子(0x10)
mov ds,cx
;索引号  00000000000_10   2
;TI  0  GDT
;权限 00 最高权限
```

 



别人写操作系统，是跟着大学的实验做，我不知道有这种方法，只知道看书：



# [ucore lab0 实验准备](https://www.cnblogs.com/whileskies/p/13138491.html)

https://www.cnblogs.com/whileskies/p/13138491.html



https://lm0963.github.io/blog/2018/05/01/%E8%BF%9B%E5%85%A5%E4%BF%9D%E6%8A%A4%E6%A8%A1%E5%BC%8F/



### 写内核

怎么写内核？我也忘记了。

编译时，设置对应选项，编译成elf文件。

内核代码，忘记了。

```assembly
; kernel.asm
[section .text]

global	_start

_start:
	mov ah, 0Fh
	mov al, 'K'
	mov gs, 0xB800
	mov [gs:(80*1+39)*2], ax
	
	jmp $
```

编译：

```shell
nasm -f elf kernel.asm -o kernel.o
ld -s kernel.o -o kernel.bin -m elf_i386
```



非常神奇。elf文件能像DOS文件一样被读入内存，然后跳转到这块内存执行指令。若是如此，就没有必要使用elf文件了。

elf文件的必要性是什么？