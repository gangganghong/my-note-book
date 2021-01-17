# 汇编和C混合使用

## 代码

### Makefile

```makefile
#######################
# Makefile for foobar #
#######################

# Programs, flags, etc.
# 定义变量,nasm，nasm汇编器
ASM		= nasm
# 定义变量，-m32，在64位电脑上用gcc编译32位代码。
CC		= gcc -m32
# -m elf_i386，在64位电脑上用ld链接32位文件。
LD		= ld -m elf_i386
# 链接器或编译器选项
ASMFLAGS	= -f elf		# 编译成elf文件？
CFLAGS		= -c				# ?
LDFLAGS		= -s				# all scripts

# This Program
BIN		= foobar
# 注意咯，可以定义多个值
OBJS		= foo.o bar.o

# All Phony Targets
# 不知道是啥意思。
.PHONY : everything final image clean realclean disasm all buildimg

# Default starting position
# 可执行命令：make everything，等价于make。
everything : $(BIN)

all : realclean everything
# 可执行命令：make final，操作是：依次执行all、clean。
final : all clean

clean :
	rm -f $(OBJS)

# 可执行命令 make clean，动作是 rm -f $(OBJS) $(BIN)。
realclean :
	rm -f $(OBJS) $(BIN)

$(BIN) : $(OBJS)
	$(LD) $(LDFLAGS) -o $(BIN) $(OBJS)

foo.o : foo.asm
	$(ASM) $(ASMFLAGS) -o $@ $<

# bar.o 是输出文件，bar.c 是输入文件。
# $(CC) 在开头定义：CC		= gcc -m32。$(CFLAGS)同理。
# $@ 是 bar.o，$<是 bar.c。
bar.o: bar.c
	$(CC) $(CFLAGS) -o $@ $<
```

Makefile的执行流程，从下到上执行，顺序如下：

1. `$(CC) $(CFLAGS) -o $@ $<`

2. `$(ASM) $(ASMFLAGS) -o $@ $<`

3. `$(LD) $(LDFLAGS) -o $(BIN) $(OBJS)`

   

### foo.asm

```assembly
; 编译链接方法
; (ld 的‘-s’选项意为“strip all”)
;
; $ nasm -f elf foo.asm -o foo.o
; $ gcc -m32 -c bar.c -o bar.o
; $ ld -m elf_i386 -s hello.o bar.o -o foobar
; $ ./foobar
; the 2nd one
; $
; extern，使本代码能使用外部文件的代码。
extern choose	; int choose(int a, int b);

[section .data]	; 数据在此
; 定义整型常量？变量？
; dd 是双字，4个字节，32位。
num1st		dd	3
num2nd		dd	4

[section .text]	; 代码在此

global _start	; 我们必须导出 _start 这个入口，以便让链接器识别
global myprint	; 导出这个函数为了让 bar.c 使用

_start:
  ; 函数调用模板而已，参数赋值入栈顺序是被调用函数的参数从右至左。
	push	dword [num2nd]	; `.
	push	dword [num1st]	;  |
	call	choose		;  | choose(num1st, num2nd);
	; 调用函数结束后，释放栈空间。
	add	esp, 8		; /

	mov	ebx, 0		; 不明白。
	mov	eax, 1		; sys_exit
	int	0x80		; 系统调用

; void myprint(char* msg, int len)
myprint:
  ; [esp] 是函数返回地址。
	mov	edx, [esp + 8]	; len
	mov	ecx, [esp + 4]	; msg
	mov	ebx, 1		; 不明白
	mov	eax, 4		; sys_write
	int	0x80		; 系统调用
	ret
```

`myprint`的函数定义，与我看到的A&T汇编不同。有时间，改成那种形式试试。我不理解目前这种写法。

### bar.c

```c
// 在这里声明，却在汇编文件中定义
void myprint(char* msg, int len);

int choose(int a, int b)
{
	if(a >= b){
		myprint("the 1st one\n", 13);
	}
	else{
		myprint("the 2nd one\n", 13);
	}

	return 0;
}
```

## 运行

```shell
make
./foobar
```



## ELF

```SHELL
[root@localhost b]# xxd -u -a -g 1 -c 16 foobar
00000000: 7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
00000010: 02 00 03 00 01 00 00 00 A0 80 04 08 34 00 00 00  ............4...
00000020: 68 10 00 00 00 00 00 00 34 00 20 00 03 00 28 00  h.......4. ...(.
00000030: 07 00 06 00 01 00 00 00 00 00 00 00 00 80 04 08  ................
00000040: 00 80 04 08 64 01 00 00 64 01 00 00 05 00 00 00  ....d...d.......
00000050: 00 10 00 00 01 00 00 00 00 10 00 00 00 A0 04 08  ................
00000060: 00 A0 04 08 08 00 00 00 08 00 00 00 06 00 00 00  ................
00000070: 00 10 00 00 51 E5 74 64 00 00 00 00 00 00 00 00  ....Q.td........
00000080: 00 00 00 00 00 00 00 00 00 00 00 00 07 00 00 00  ................
00000090: 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
000000a0: FF 35 04 A0 04 08 FF 35 00 A0 04 08 E8 24 00 00  .5.....5.....$..
000000b0: 00 83 C4 08 BB 00 00 00 00 B8 01 00 00 00 CD 80  ................
000000c0: 8B 54 24 08 8B 4C 24 04 BB 01 00 00 00 B8 04 00  .T$..L$.........
000000d0: 00 00 CD 80 C3 55 89 E5 83 EC 08 8B 45 08 3B 45  .....U......E.;E
000000e0: 0C 7C 14 83 EC 08 6A 0D 68 10 81 04 08 E8 CE FF  .|....j.h.......
000000f0: FF FF 83 C4 10 EB 12 83 EC 08 6A 0D 68 1D 81 04  ..........j.h...
00000100: 08 E8 BA FF FF FF 83 C4 10 B8 00 00 00 00 C9 C3  ................
00000110: 74 68 65 20 31 73 74 20 6F 6E 65 0A 00 74 68 65  the 1st one..the
00000120: 20 32 6E 64 20 6F 6E 65 0A 00 00 00 14 00 00 00   2nd one........
00000130: 00 00 00 00 01 7A 52 00 01 7C 08 01 1B 0C 04 04  .....zR..|......
00000140: 88 01 00 00 1C 00 00 00 1C 00 00 00 89 FF FF FF  ................
00000150: 3B 00 00 00 00 41 0E 08 85 02 42 0D 05 77 C5 0C  ;....A....B..w..
00000160: 04 04 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000170: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
```



1. 第一行

   1. `7F 45 4C 46`, 是固定的elf魔数，表示 0x7F、E、L、F。
   2. 01，32位elf文件。
   3. 01，小端字节序。
   4. 01，当前版本。

2. 第二行：`02 00 03 00 01 00 00 00 A0 80 04 08 34 00 00 00`

   1. 0x0002，e_type，值为2表示ET_EXEC，可执行文件。
   2. 03 00，0x0003，e_machine，EM_386，表示该elf文件是运行在Intel 80386平台。
   3. 01 00 00 00，0x00000001，e_version，版本信息。
   4. A0 80 04 08，0x080480A0，e_entry，程序的虚拟入口地址。
   5. 34 00 00 00，0x00000034，e_phoff，program header table 程序头表在文件中的偏移量。

3. 第三行：`68 10 00 00 00 00 00 00 34 00 20 00 03 00 28 00`

   1. 68 10 00 00，4字节，0x00001068，e_shoff，section header table 节头表在文件内的偏移量。若没有节头表，该项为0。
   2. 00 00 00 00，4字节，0x00000000，e_flags，
   3. 34 00，2字节，0x0034，e_ehsize，elf header size，elf的header的大小是0x34，与e_phoff的值一致，elf后紧跟program header table。
   4. 20 00，2字节，0x0020，e_phentsize，即 program header的结构：struct Elf32_Phdr的字节大小，值为 0x20 字节。
   5. 03 00，2字节，0x0003，e_phnum，程序头表中段的个数，3个段。
   6. 28 00，2字节，0x0028，e_shentsize，节头表中各个节的大小。

4. 第四行：07 00 06 00 01 00 00 00 00 00 00 00 00 80 04 08

   1. 07 00，0x0007，2字节，e_shnum，节头表中节的个数，

   2. 06 00，0x0006，2字节，e_shstrndex，string name table在节头表中的索引。

   3. 分析 struct Elf32_Phdr结构，

      1. ```shell
         00000030: 07 00 06 00 01 00 00 00 00 00 00 00 00 80 04 08  ................
         00000040: 00 80 04 08 64 01 00 00 64 01 00 00 05 00 00 00  ....d...d.......
         00000050: 00 10 00 00 01 00 00 00 00 10 00 00 00 A0 04 08  ................
         00000060: 00 A0 04 08 08 00 00 00 08 00 00 00 06 00 00 00  ................
         00000070: 00 10 00 00 51 E5 74 64 00 00 00 00 00 00 00 00  ....Q.td........
         00000080: 00 00 00 00 00 00 00 00 00 00 00 00 07 00 00 00  ................
         ```

      2. 01 00 00 00，p_type，PT_LOAD类型，可加载程序段

      3. 00 00 00 00 ，p_offset，本段在文件内的偏移量。

      4. 00 80 04 08，p_vaddr，本段被加载到内存后的起始虚拟地址。

      5. 00 80 04 08，p_paddr，通常和 p_vaddr 一致，保留项，不必关注。

      6. 64 01 00 00，p_filesz，本段在文件中的字节大小。

      7. 64 01 00 00，p_memsz，本段在内存中的字节大小。无论是在文件中还是在内存中，段本身的大小不会变，所以，p_memsz 等于 p_filesz。

      8. 05 00 00 00，p_flags，与本段相关的标志，5 = 4 + 1 = PF_R + PF_X，可读、可执行，据此推断，此段为代码段。

      9. 00 10 00 00，p_align，本段对齐方式。

      10. 上面是对第一个段的分析，刚好32个字节，即0x20个字节。第一行的后12个字节 + 第二行的16个字节 + 第三行的前4个字节 = 32个字节。

### 实例分析

```c
int main(void){
        while(1);

        return 0;
}
```



```shell
gcc -m32 -c -o while.o while.c
ld -m elf_i386 while.o -Ttext 0xc0001500 -e main -o while.bin
```

```shell
[root@localhost a]# xxd -u -a -g 1 -c 16 while.bin
00000000: 7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
00000010: 02 00 03 00 01 00 00 00 00 15 00 C0 34 00 00 00  ............4...
00000020: 88 06 00 00 00 00 00 00 34 00 20 00 02 00 28 00  ........4. ...(.
00000030: 07 00 06 00 01 00 00 00 00 00 00 00 00 10 00 C0  ................
00000040: 00 10 00 C0 3C 05 00 00 3C 05 00 00 05 00 00 00  ....<...<.......
00000050: 00 10 00 00 51 E5 74 64 00 00 00 00 00 00 00 00  ....Q.td........
00000060: 00 00 00 00 00 00 00 00 00 00 00 00 06 00 00 00  ................
00000070: 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000080: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
*
00000500: 55 89 E5 EB FE 00 00 00 14 00 00 00 00 00 00 00  U...............
00000510: 01 7A 52 00 01 7C 08 01 1B 0C 04 04 88 01 00 00  .zR..|..........
00000520: 18 00 00 00 1C 00 00 00 D8 FF FF FF 05 00 00 00  ................
00000530: 00 41 0E 08 85 02 42 0D 05 00 00 00 47 43 43 3A  .A....B.....GCC:
00000540: 20 28 47 4E 55 29 20 38 2E 33 2E 31 20 32 30 31   (GNU) 8.3.1 201
00000550: 39 31 31 32 31 20 28 52 65 64 20 48 61 74 20 38  91121 (Red Hat 8
00000560: 2E 33 2E 31 2D 35 29 00 00 00 00 00 00 00 00 00  .3.1-5).........
00000570: 00 00 00 00 00 00 00 00 00 00 00 00 00 15 00 C0  ................
00000580: 00 00 00 00 03 00 01 00 00 00 00 00 08 15 00 C0  ................
00000590: 00 00 00 00 03 00 02 00 00 00 00 00 00 00 00 00  ................
```



1. 第一行，7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00

   1. 7F 45 4C 46，elf 文件的固定魔数。
   2. 01，32位elf文件。
   3. 01，小端字节序。
   4. 01，当前版本。

2. 第二行，02 00 03 00 01 00 00 00 00 15 00 C0 34 00 00 00

   1. 0x0002，e_type，2字节，值为2表示ET_EXEC，可执行文件。
   2. 03 00，0x0003，e_machine，2字节，EM_386，表示该elf文件是运行在Intel 80386平台。
   3. 01 00 00 00，0x00000001，e_version，4字节，版本信息。
   4. 00 15 00 C0，0xC0001500，e_entry，程序的虚拟入口地址。
   5. 34 00 00 00，0x34，e_phoff，program header table 程序头表在文件中的偏移量。

3. 第三行，88 06 00 00 00 00 00 00 34 00 20 00 02 00 28 00 

   1. 88 06 00 00，4字节，0x00000688，e_shoff，section header table 节头表在文件内的偏移量。若没有节头表，该项为0。
   2. 00 00 00 00，4字节，0x00000000，e_flags，
   3. 34 00，2字节，0x0034，e_ehsize，elf header size，elf的header的大小是0x34，与e_phoff的值一致，elf后紧跟program header table。
   4. 20 00，2字节，0x0020，e_phentsize，即 program header的结构：struct Elf32_Phdr的字节大小，值为 0x20 字节。
   5. 02 00，2字节，0x0003，e_phnum，程序头表中段的个数，2个段。
   6. 28 00，2字节，0x0028，e_shentsize，节头表中各个节的大小。

4. 第四行，07 00 06 00 01 00 00 00 00 00 00 00 00 10 00 C0

   1. 07 00，0x0007，2字节，e_shnum，节头表中节的个数，

   2. 06 00，0x0006，2字节，e_shstrndex，string name table在节头表中的索引。

   3. 分析 struct Elf32_Phdr结构。2个段，每个段0x20个字节，总计0x40个字节 = 64个字节。每行16个字节，共4行（4、5、6、7、8）。

      1. ```shell
         00000030: 07 00 06 00 01 00 00 00 00 00 00 00 00 10 00 C0  ................
         00000040: 00 10 00 C0 3C 05 00 00 3C 05 00 00 05 00 00 00  ....<...<.......
         00000050: 00 10 00 00 51 E5 74 64 00 00 00 00 00 00 00 00  ....Q.td........
         00000060: 00 00 00 00 00 00 00 00 00 00 00 00 06 00 00 00  ................
         ```

      2. 01 00 00 00，p_type，PT_LOAD类型，可加载程序段

      3. 00 00 00 00 ，p_offset，本段在文件内的偏移量。

      4. 00 10 00 C0，p_vaddr，本段被加载到内存后的起始虚拟地址。

      5. 00 80 04 08，p_paddr，通常和 p_vaddr 一致，保留项，不必关注。

      6. 64 01 00 00，p_filesz，本段在文件中的字节大小。

      7. 64 01 00 00，p_memsz，本段在内存中的字节大小。无论是在文件中还是在内存中，段本身的大小不会变，所以，p_memsz 等于 p_filesz。

      8. 05 00 00 00，p_flags，与本段相关的标志，5 = 4 + 1 = PF_R + PF_X，可读、可执行，据此推断，此段为代码段。

      9. 00 10 00 00，p_align，本段对齐方式。

      10. 上面是对第一个段的分析，刚好32个字节，即0x20个字节。第一行的后12个字节 + 第二行的16个字节 + 第三行的前4个字节 = 32个字节。

5. 分析：

   1. 00 10 00 C0，p_vaddr，0xC0001000，

   2. 0xC0001500，e_entry，

   3. 00 00 00 00 ，p_offset，本段在文件内的偏移量是0。

   4. 程序的入口地址在该段内的偏移量是：0xC0001500 - 0xC0001000。

   5. 程序的入口地址在文件内的偏移量是：段在文件内的偏移量 + 入口地址在段内的偏移量 = 0 + 0xC0001500 - 0xC0001000 = 0x0000500 = 1280。

      1. 很简单问题，但我从小就感觉这种细小的边界问题有点棘手，差是应该加1还是减去1还是不加不减。

   6. 查看 while.bin 从1280开始的5个字节的内容：

      1. ```shell
         [root@localhost a]# xxd -u -a -g 1 -c 16 -l 5 -s 1280 while.bin
         00000500: 55 89 E5 EB FE                                   U....
         [root@localhost a]#
         ```

      2. ```assembly
                 .file   "while.c"
                 .text
                 .globl  main
                 .type   main, @function
         main:
         .LFB0:
                 .cfi_startproc
                 pushl   %ebp
                 .cfi_def_cfa_offset 8
                 .cfi_offset 5, -8
                 movl    %esp, %ebp
                 .cfi_def_cfa_register 5
         .L2:
                 jmp     .L2
                 .cfi_endproc
         .LFE0:
                 .size   main, .-main
                 .ident  "GCC: (GNU) 8.3.1 20191121 (Red Hat 8.3.1-5)"
                 .section        .note.GNU-stack,"",@progbits
         ```

      3. 机器码：

         1. 0X55：pushl   %ebp
         2. 0X89E5：movl    %esp, %ebp
         3. 0XEBFE：jmp     .L2

      4. 验证：

         1. ```shell
            # 在64位机器上编译32位汇编代码(--32，必须放在前面），并加上调试信息(-g)
            [root@localhost a]# as --32 -g -o while.o while.s
            [root@localhost a]# file while.o
            while.o: ELF 32-bit LSB relocatable, Intel 80386, version 1 (SYSV), with debug_info, not stripped
            # 在64位机器上连接32目标文件(-m elf_i386），并指定程序入口地址（-Ttext）和入口函数(-e)。
            [root@localhost a]# ld -m elf_i386 while.o -Ttext 0xc0001500 -e main -o while.bin
            [root@localhost a]# file while.bin
            while.bin: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), statically linked, with debug_info, not stripped
            [root@localhost a]#
            ```

         2. ```shell
            [root@localhost a]# lldb ./while.bin
            (lldb) target create "./while.bin"
            Current executable set to '/home/cg/os/asm2/chapter5/a/while.bin' (i386).
            (lldb) breakpoint set --file while.s --line 10
            Breakpoint 1: where = while.bin`main, address = 0xc0001500
            (lldb) disassemble -b -c 5
            error: Cannot disassemble around the current function without a selected frame.
            
            (lldb) run
            Process 60886 launched: '/home/cg/os/asm2/chapter5/a/while.bin' (i386)
            Process 60886 stopped
            * thread #1, name = 'while.bin', stop reason = signal SIGSTOP
                frame #0: 0xc0001503 while.bin`main at while.s:16
               13  		movl	%esp, %ebp
               14  		.cfi_def_cfa_register 5
               15  	.L2:
            -> 16  		jmp	.L2
               17  		.cfi_endproc
               18  	.LFE0:
               19  		.size	main, .-main
            (lldb) disassemble -b -c 5
            while.bin`main:
                0xc0001500 <+0>: 55        pushl  %ebp
                0xc0001501 <+1>: 89 e5     movl   %esp, %ebp
            ->  0xc0001503 <+3>: eb fe     jmp    0xc0001503                ; <+3>
                0xc0001505:      00 00     addb   %al, (%eax)
                0xc0001507:      00 14 00  addb   %dl, (%eax,%eax)
            (lldb)
            ```

            1. 使用 `disassemble -b -c 5` 能查看汇编代码对应的机器码。`-b` 输出机器码，`-c 5` 输出5行机器码。
               1. 也许是因为这个程序是个没有其他指令的死循环，需要遵循 `run->ctrl+c->disassemble -b -c 5` 才能打印出机器码。
            2. 前面用xxd查看1280字节开始的5个字节是 `00000500: 55 89 E5 EB FE `，与这里查看的结果，是一致的。

## 工具

### lldb

帮助资料：

```html
https://lldb.llvm.org/lldb-gdb.html
(lldb) help memory
(lldb) help disassemble
(lldb) help memory write
http://armconverter.com/
```



在bochs中断点调试时，显示的汇编代码，基本与文件中的汇编代码一致，但是与反汇编出来的汇编代码不一致。

反汇编回来的代码，和用xxd查看的代码，是一致的。

今天，在这个问题上耗费了七八个小时。时间，不是耗费在bochs调试方法，也不是linux命令，也不是任何新知识，而是耗费在重复执行错误的命令。这是我的老毛病，像进入死循环一样，不知道停下来。

我因为看不懂书中的汇编代码而想调试。在bochs中调试，对我理解代码没有作用，看到的全是十六进数据。

```
<bochs:7> xp /1wx 0x00080034
[bochs]:
0x0000000000080034 <bogus+       0>:	0x00000001
<bochs:8> xp /1wx 0x00080042
[bochs]:
0x0000000000080042 <bogus+       0>:	0x04100003
<bochs:9> xp /1wx 0x0008003c
[bochs]:
0x000000000008003c <bogus+       0>:	0x00030000
<bochs:10> 
```



```
[root@localhost e]# xxd -u -a -g 1 -c 16  kernel.bin
00000000: 7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
00000010: 02 00 03 00 01 00 00 00 00 04 03 00 34 00 00 00  ............4...
00000020: 24 04 00 00 00 00 00 00 34 00 20 00 01 00 28 00  $.......4. ...(.
00000030: 03 00 02 00 01 00 00 00 00 00 00 00 00 00 03 00  ................
00000040: 00 00 03 00 10 04 00 00 10 04 00 00 05 00 00 00  ................
00000050: 00 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000060: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
*
00000400: 66 87 DB B4 0F B0 4B 65 66 A3 EE 00 00 00 EB FE  f.....Kef.......
00000410: 00 2E 73 68 73 74 72 74 61 62 00 2E 74 65 78 74  ..shstrtab..text
00000420: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000430: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000440: 00 00 00 00 00 00 00 00 00 00 00 00 0B 00 00 00  ................
00000450: 01 00 00 00 06 00 00 00 00 04 03 00 00 04 00 00  ................
00000460: 10 00 00 00 00 00 00 00 00 00 00 00 10 00 00 00  ................
00000470: 00 00 00 00 01 00 00 00 03 00 00 00 00 00 00 00  ................
00000480: 00 00 00 00 10 04 00 00 11 00 00 00 00 00 00 00  ................
00000490: 00 00 00 00 01 00 00 00 00 00 00 00              ............
```

第5行的最后四个字节是 00 00 03 00，小端法，转为人眼格式，0x00030000，与 bochs 中的打印数据吻合。

```makefile
ld -s -Ttext 0x30400 -o $@ $(subst .asm,.o,$(KERNEL))#Ttext指定程序入口地址，是虚拟内存地址
```

指定了入口地址，会自动设置好 p_vaddr。只需要将程序头复制到 p_vaddr，就可以了。至于为什么这样就可以了，我不知道。也许，并不需要知道吧。

复制，是从物理地址处复制到虚拟地址处。很奇怪，

readelf -h 读取ELF可执行文件头

readelf -S 查看ELF文件Section 信息

objdump -d 看目标文件汇编代码

```
[root@localhost e]# ld --help | grep 'Ttext'
  -Ttext ADDRESS              Set address of .text section
  -Ttext-segment ADDRESS      Set address of text segment
```

## 理解C语言的一些语法

 jmpf 0xf000:e05b          ; ea5be000f0

rsi: 00000000_200e2305

es:0x0010, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
cs:0x0008, dh=0x00cf9b00, dl=0x0000ffff, valid=1
	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
ss:0x0010, dh=0x00cf9300, dl=0x0000ffff, valid=31



================

rax: 00000000_00060000
rbx: 00000000_00000400
rcx: 00000000_00005f2c
rdx: 00000000_00000000
rsp: 00000000_000916f9
rbp: 00000000_00091705
rsi: 00000000_00060000
rdi: 00000000_00030000
r8 : 00000000_00000000
r9 : 00000000_00000000
r10: 00000000_00000000
r11: 00000000_00000000
r12: 00000000_00000000
r13: 00000000_00000000
r14: 00000000_00000000
r15: 00000000_00000000
rip: 00000000_00090482



<bochs:3> sreg
es:0x0010, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
cs:0x0008, dh=0x00cf9b00, dl=0x0000ffff, valid=1
	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
ss:0x0010, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
ds:0x0010, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
fs:0x0010, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
gs:0x001b, dh=0x0000f30b, dl=0x8000ffff, valid=7
	Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0000, dh=0x00008200, dl=0x0000ffff, valid=1
tr:0x0000, dh=0x00008b00, dl=0x0000ffff, valid=1
gdtr:base=0x000000000009013d, limit=0x1f
idtr:base=0x0000000000000000, limit=0x3ff

============

<bochs:2> r
CPU0:
rax: 00000000_534d6000
rbx: 00000000_00000000
rcx: 00000000_0000002e
rdx: 00000000_534d000c
rsp: 00000000_000000fe
rbp: 00000000_0000029e
rsi: 00000000_000e029d
rdi: 00000000_0000007a
r8 : 00000000_00000000
r9 : 00000000_00000000
r10: 00000000_00000000
r11: 00000000_00000000
r12: 00000000_00000000
r13: 00000000_00000000
r14: 00000000_00000000
r15: 00000000_00000000
rip: 00000000_00000233

rax: 00000000_00060000
rbx: 00000000_00000400
rcx: 00000000_00005f2c
rdx: 00000000_00000000
rsp: 00000000_00091719
rbp: 00000000_00091725
rsi: 00000000_00060000
rdi: 00000000_00030000
r8 : 00000000_00000000
r9 : 00000000_00000000
r10: 00000000_00000000
r11: 00000000_00000000
r12: 00000000_00000000
r13: 00000000_00000000
r14: 00000000_00000000
r15: 00000000_00000000
rip: 00000000_000904a2

<bochs:9> sreg
es:0x0010, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
cs:0x0008, dh=0x00cf9b00, dl=0x0000ffff, valid=1
	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
ss:0x0010, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
ds:0x0010, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
fs:0x0010, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
gs:0x001b, dh=0x0000f30b, dl=0x8000ffff, valid=7
	Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0000, dh=0x00008200, dl=0x0000ffff, valid=1
tr:0x0000, dh=0x00008b00, dl=0x0000ffff, valid=1
gdtr:base=0x000000000009013d, limit=0x1f
idtr:base=0x0000000000000000, limit=0x3ff

xp /1wx 00091725

xp /1wx  0x0009172D

xp /1wx 0x00091731

xp /1wx 0x00091735

<bochs:14> xp /1wx  0x0009172D
[bochs]:
0x000000000009172d <bogus+       0>:	0x00030000
<bochs:17> xp /1wx 0x00091731
[bochs]:
0x0000000000091731 <bogus+       0>:	0x00060000
<bochs:18> xp /1wx 0x00091735
[bochs]:
0x0000000000091735 <bogus+       0>:	0x00005f2c

<bochs:6> xp /1wx  0x0009172D
[bochs]:
0x000000000009172d <bogus+       0>:	0x054b4b83
<bochs:7> xp /1wx 0x00091731
[bochs]:
0x0000000000091731 <bogus+       0>:	0x200e2305
<bochs:8> xp /1wx 0x00091735
[bochs]:
0x0000000000091735 <bogus+       0>:	0x02040200



========

<bochs:2> reg
CPU0:
rax: 00000000_534d6000
rbx: 00000000_00000000
rcx: 00000000_0000002e
rdx: 00000000_534d000c
rsp: 00000000_000000fe
rbp: 00000000_0000029e
rsi: 00000000_000e029d
rdi: 00000000_0000007a
r8 : 00000000_00000000
r9 : 00000000_00000000
r10: 00000000_00000000
r11: 00000000_00000000
r12: 00000000_00000000
r13: 00000000_00000000
r14: 00000000_00000000
r15: 00000000_00000000
rip: 00000000_00000233
eflags 0x00000006: id vip vif ac v

<bochs:3> sreg
es:0x6000, dh=0x00009306, dl=0x0000ffff, valid=3
	Data segment, base=0x00060000, limit=0x0000ffff, Read/Write, Accessed
cs:0x9000, dh=0x00009309, dl=0x0000ffff, valid=1
	Data segment, base=0x00090000, limit=0x0000ffff, Read/Write, Accessed
ss:0x9000, dh=0x00009309, dl=0x0000ffff, valid=7
	Data segment, base=0x00090000, limit=0x0000ffff, Read/Write, Accessed
ds:0x9000, dh=0x00009309, dl=0x0000ffff, valid=7
	Data segment, base=0x00090000, limit=0x0000ffff, Read/Write, Accessed
fs:0x0000, dh=0x00009300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
gs:0x0000, dh=0x00009300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0000, dh=0x00008200, dl=0x0000ffff, valid=1
tr:0x0000, dh=0x00008b00, dl=0x0000ffff, valid=1
gdtr:base=0x00000000000f9af7, limit=0x30
idtr:base=0x0000000000000000, limit=0x3ff

<bochs:6> xp /1wx  0x0009172D
[bochs]:
0x000000000009172d <bogus+       0>:	0x054b4b83
<bochs:7> xp /1wx 0x00091731
[bochs]:
0x0000000000091731 <bogus+       0>:	0x200e2305
<bochs:8> xp /1wx 0x00091735
[bochs]:
0x0000000000091735 <bogus+       0>:	0x02040200

rsp: 00000000_0009172d

xp  /1wx 0x0009172d

xp  /1wx 0x00091731

xp  /1wx 0x00091735

<bochs:10> xp  /1wx 0x0009172d
[bochs]:
0x000000000009172d <bogus+       0>:	0x054b4b83
<bochs:11> xp  /1wx 0x00091731
[bochs]:
0x0000000000091731 <bogus+       0>:	0x200e2305

<bochs:12> xp  /1wx 0x00091735
[bochs]:
0x0000000000091735 <bogus+       0>:	0x02040200

-------------------------------------------

<bochs:5> xp  /1wx 0x0009172d
[bochs]:
0x000000000009172d <bogus+       0>:	0x00030000
<bochs:6> xp  /1wx 0x00091731
[bochs]:
0x0000000000091731 <bogus+       0>:	0x00060000
<bochs:7> xp  /1wx 0x00091735
[bochs]:
0x0000000000091735 <bogus+       0>:	0x00005f2c

 interrupt(): gate descriptor is not valid sys seg (vector=0x0e)
00045883472e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x08)
00045883472i[CPU0  ] CPU is in protected mode (active)
00045883472i[CPU0  ] CS.mode = 32 bit
00045883472i[CPU0  ] SS.mode = 32 bit
00045883472i[CPU0  ] EFER   = 0x00000000
00045883472i[CPU0  ] | EAX=200e2305  EBX=00000400  ECX=02040200  EDX=00000000
00045883472i[CPU0  ] | ESP=00091719  EBP=00091725  ESI=200e2305  EDI=054b4b83
00045883472i[CPU0  ] | IOPL=0 id vip vif ac vm RF nt of df if tf sf zf af PF cf
00045883472i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
00045883472i[CPU0  ] |  CS:0008( 0001| 0|  0) 00000000 ffffffff 1 1
00045883472i[CPU0  ] |  DS:0010( 0002| 0|  0) 00000000 ffffffff 1 1
00045883472i[CPU0  ] |  SS:0010( 0002| 0|  0) 00000000 ffffffff 1 1
00045883472i[CPU0  ] |  ES:0010( 0002| 0|  0) 00000000 ffffffff 1 1
00045883472i[CPU0  ] |  FS:0010( 0002| 0|  0) 00000000 ffffffff 1 1
00045883472i[CPU0  ] |  GS:001b( 0003| 0|  3) 000b8000 0000ffff 0 0
00045883472i[CPU0  ] | EIP=000904a2 (000904a2)
00045883472i[CPU0  ] | CR0=0xe0000011 CR2=0x200e2305
00045883472i[CPU0  ] | CR3=0x00200000 CR4=0x00000000
(0).[45883472] [0x0000000904a2] 0008:00000000000904a2 (unk. ctxt): mov al, byte ptr ds:[esi] ; 3e8a06
00045883472e[CPU0  ] exception(): 3rd (13) exception with no resolution, shutdown status is 00h, resetting
00045883472i[SYS   ] bx_pc_system_c::Reset(HARDWARE) called
00045883472i[CPU0  ] cpu hardware reset
00045883472i[APIC0 ] allocate APIC id=0 (MMIO enabled) to 0x0000fee00000
00045883472i[CPU0  ] CPU[0] is the bootstrap processor

xp /1wx 0x00063806

xp /1wx 0x0006380A          ;		0x20082305

### 从boot读取软盘扇区中的汇编

```assembly
;NASM 汇编
;nasm this.asm -o hello_os
org 07c00h
    mov ax, cs
    mov ds, ax
call welcome_words
call load_os
jmp os
welcome_words:
	mov ax,boot_message
	mov bp,ax                             ;bp存储需要显示的字符串的起始地址
	mov cx,boot_message_length            ;cx存储要显示的字符串的长度
	mov ax,01301h                         ;ah=13h,是int 10h中断的参数之一,al=01h,标识输出方式
	mov bx,000ah                          ;bh为页码,bl为颜色
	mov dx,0d00h                          ;dx为显示位置坐标,0d行,0列
	int 10h
	ret
load_os:
	mov ah,02h                            ;读磁盘扇区
	mov al,01h                            ;读取1个扇区
	mov ch,00h                            ;起始磁道
	mov cl,02h                            ;起始扇区
	mov dh,00h                            ;磁头号
	mov dl,00h                            ;驱动器号
	mov bx,os                             ;存储缓冲区
	int 13h
	ret
boot_message:
	db "[Boot]modu os"
	db 0dh,0ah                            ;换行
	db "[Boot]loading..."
boot_message_length equ $-boot_message    ;计算字符串长度
times 510-($-$$) db 0                     ;填充至510byte
dw 0xaa55                                 ;引导扇区标识,至此文件大小为512byte

os:
	call os_say_hello
	jmp $
os_say_hello:
	mov ax,os_message
	mov bp,ax
	mov cx,os_message_length
	mov ax,01301h
	mov bx,000eh
	mov dx,1000h
	int 10h
	ret
os_message:
	db "[OS]os loaded"
	db 0dh,0ah
	db "[OS]happy using"
os_message_length equ $-os_message
times 1022-($-$$) db 0
```



没能力用bochs运行它。价值在于，读取软盘。

#### 从磁盘(扇区2到18)上读取数据到内存中

## 下面代码读取柱面:0,磁头:0,扇区从2到18的数据到内存 0x8200~0xa3ff处

- 需要明白以下几点:

  - 给定柱面,磁头,一个扇形区域是512字节,对应的物理可以理解为512个灯泡组(一个灯泡组有8个小灯泡)

  - 确定读取到内存中的位置

    - 为什么是0x8200:因为0x8000~0x81ff这512个字节要留给启动区.
    - 为什么是0x8000以后,因为这一段内存区域,很少有人使用,故读取到这段内存上出错的机率低

  - CH(计数寄存器的高位)用于存储柱面信息

  - DH(数据寄存器的高位)用于存储磁头信息

  - CL(计数寄存器的低位)存储扇区

  - SI(源变址寄存器)用于存储读取磁盘失败的次数

  - 根据BIOS提供的信息：

    - AH = 0x02 ; 读入磁盘
    - AL = 1 ; 一次读取1个扇区

  - 系统复位: 复位软盘状态,再读一次

    ```
    MOV		AH,0x0820
    MOV		DL,0x00
    INT			0x13
    123
    ```

  - SI大于5时,执行error代码段

    - JAE(Jump if above or equal):大于等于

    ```
    CMP		SI,5
    JAE		error
    12
    ```

  - JNC(Jump if not carry):如果没有出错的话跳到后面的代码段

  - next代码段:用于读取下一个磁盘扇形区到内存中

    - 一个扇形区域是512B,对应的段地址(es)偏移为0x0020,故使用AX给es加0x0020



```
; haribote-ipl
; TAB=4

		ORG		0x7c00			; 程序从哪里装入

; 以下是对标准FAT12格式软盘的描述

		JMP		entry
		DB		0x90
		DB		"HARIBOTE"		; 可以自由书写引导扇形区的名称 (8字节)
		DW		512				; 1扇区的大小 (必须做成512)
		DB 		1 				; 集群大小 (必须设置在一个扇区)
		DW		1				; FAT从哪里开始 (一般从第一个部分开始)
		DB		2				; FAT的个数 (必须是2)
		DW		224				; 根目录区域的大小 (一般为224条目)
		DW 		2880			; 这个驱动器的大小 (必须是2880扇区)
		DB		0xf0			; 媒体类型 (必须是0xf0)
		DW		9				; FAT区域的长度 (必须设置为9个扇区)
		DW		18				; 1卡车有几个扇区 (必须是18)
		DW		2				; 头数 (必须为2)
		DD		0				; 因为不使用分区, 这里一定0
		DD 		2880			; 再写一次这个驱动器的大小
		DB		0,0,0x29		; 预先设置值
		DD		0xffffffff		; 音量序列号
		DB		"HARIBOTEOS "	; 磁盘名称 (11字节)
		DB		"FAT12   "		; 格式名称 (8字节)
		RESB	18				; 暂且空开18字节

; 程序主体

entry:
		MOV		AX,0			; 寄存器初始化
		MOV 	SS,AX
		MOV		SP,0x7c00
		MOV		DS,AX

; 读磁盘

		MOV		AX,0x0820
		MOV		ES,AX
		MOV		CH,0			; 柱面0
		MOV		DH,0			; 磁头0 (正面)
		MOV 	CL,2			; 扇区2
readloop:						; 清零失败寄存器
		MOV 	SI,0			; 记录失败次数的寄存器

; 重新尝试
retry:
		MOV 	AH,0x02			; AH=0x02 : 读入磁盘(柱面0,磁头0,扇区2)
		MOV 	AL,1			; 1个扇区
		MOV 	BX,0
		MOV		DL,0x00			; A驱动器
		INT		0X13			; 调用磁盘BIOS	
		JNC		next			; 没出错的话跳转到next
		ADD 	SI,1			; 出错了,SI加1
		CMP		SI,5			; 比较SI与5
		JAE		error			; SI >=5时, 跳转到error
		; 复位软盘状态
		MOV		AH,0x00
		MOV		DL,0x00			; A驱动器
		INT		0x13			; 重置驱动器
		JMP 	entry
	
; 读取下一个扇区
; CL:扇区号, ES:读入的地址
next:
		; 把内存地址后移0x200
		MOV		AX,ES			
		ADD		AX,0x0020
		MOV 	ES,AX			; ES无法直接加 0x020
		ADD		CL,1			; 往CL里加1
		
		; 比较CL与18,如果小于18则跳转到readloop
		CMP		CL,18			
		JBE		readloop


fin:	
		HLT						; 让CPU停止, 等待指令
		JMP		fin				; 无限循环
		
error:	
		MOV		SI,msg
putloop:
		MOV		AL,[SI]
		ADD		SI,1			; 给SI加1
		CMP		AL,0
		
		JE		fin
		MOV		AH,0x0e			; 显示一个文字
		MOV		BX,15			; 指定字符颜色
		INT		0x10			; 调用显卡BIOS
		JMP		putloop
msg:
		DB		0x0a, 0x0a		; 换行2次
		DB		"load error"	;
		DB		0x0a			; 换行
		DB		0
		
		RESB	0x7dfe-$		; 用0x00将代码不全至 0x7dfe-$
		
		DB		0x55, 0xaa
```



````
00022651756p[BIOS  ] >>PANIC<< set_diskette_current_cyl(): drive > 1
00022653071i[CPU0  ] WARNING: HLT instruction with IF=0!
````

xp /1wx 0x9000:0x010a

## 解析硬盘参数

````
(gdb) p hdbuf
$2 = "@\000\242\000\000\000\020\000\000~\000\002?\000\000\000\000\000\000\000XBDH0010 1  @\000\242\000\000\000\020\000\000~\000\002?\000\000\000\000\000\000\000XBDH0010 1  43", ' ' <repeats 28 times>, "\020\000\001\000\000\003\000\000\000\002\000\002\a\000\242\000\020\000?\000\340}\002\000\000\000\340}\002\000\000\000\a\000\000\000x\000x\000x\000x", '\000' <repeats 23 times>, "~\000\000\000\000@\000t\000@\000@\000t\000@?\000\000\000\000\000\000\000\000\000\001`", '\000' <repeats 12 times>...
````



怎么解析这种数据啊？

