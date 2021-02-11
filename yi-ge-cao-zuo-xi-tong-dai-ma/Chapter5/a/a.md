# hello

```assembly
; 编译链接方法
; (ld 的‘-s’选项意为“strip all”)
;
; $ nasm -f elf hello.asm -o hello.o
; $ ld -s hello.o -o hello
; 在64位机器上应该使用：ld -m elf_i386 -s hello.o -o hello
; $ ./hello
; Hello, world!
; $

[section .data]	; 数据在此

; 0Ah 是何意?在手册找了大概五六分钟，没找到。更改为0Fh、01h，都能正常运行。
; 定义一个字符串，strHello相当于变量名。"Hello, world!" 是变量值。db是变量占用的空间，一个字节？
strHello	db	"Hello, world!", 0Ah
; strHello 表示这个数据的内存起始地址，$是strHello表示的数据的内存的结束地址。
STRLEN		equ	$ - strHello

[section .text]	; 代码在此

global _start	; 我们必须导出 _start 这个入口，以便让链接器识别

; 这是在定义一个函数，可是与A&t汇编不同。
; 我不理解。
; 1>mov	ebx, 1
_start:
  ; 这两句应该是给sys_write的参数赋值，但是，参数赋值不是使用栈吗？
	mov	edx, STRLEN
	mov	ecx, strHello
	mov	ebx, 1		; 注释掉这句也没问题。改成 mov ebx, 10不行。
	mov	eax, 4		; sys_write
	int	0x80		; 系统调用
	mov	ebx, 0
	mov	eax, 1		; sys_exit
	int	0x80		; 系统调用
```

## 运行

```shell
nasm -f elf -o hello.o hello.asm
ld -m elf_i386 -s hello.o -o hello
./hello
```

其他调试信息

```shell
[root@localhost a]# nasm -f elf -o hello.o hello.asm
[root@localhost a]# file hello.o
hello.o: ELF 32-bit LSB relocatable, Intel 80386, version 1 (SYSV), not stripped
[root@localhost a]# ld -m elf_i386 -s hello.o -o hello
[root@localhost a]# file hello
hello: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), statically linked, stripped
[root@localhost a]# ./hello
Hello, world![root@localhost a]#
```



## 错误

```assembly
[root@localhost a]# ls
hello.asm  hello.o
[root@localhost a]# ld -s hello.o -o hello
ld: i386 architecture of input file `hello.o' is incompatible with i386:x86-64 output
[root@localhost a]#
```

解决：`ld -m elf_i386 -s hello.o -o hello` 。



