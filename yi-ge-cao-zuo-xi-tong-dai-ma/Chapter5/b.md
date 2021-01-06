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

      10. 上面第对第一个段的分析，刚好32个字节，即0x20个字节。第一行的后12个字节 + 第二行的16个字节 + 第三行的前4个字节 = 32个字节。

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