# 内核雏形

## 在Linux下用汇编写Hello World

编译链接方法：

```shell
nasm -f elf hello.asm -o hello.o
# -m elf_i386 ，在64位系统上编译出32位程序需要使用
ld -s hello.o -o hello -m elf_i386
```

代码：

```assembly
[section .data]
strHello	db	"Hello, OS World!"
strHelloLenght	equ $ - strHello

[section .text]
global _start

_start:
	mov eax, 4
	mov ebx, 1
	mov ecx, strHello
	mov edx, strHelloLength
	int 0x80
	mov eax, 1
	mov ebx, 0
	int 0x80
```

在linux下使用汇编编程，使用系统调用的套路，掌握两个知识点：

1. 使用中断`int 0x80`，并给这个中断提供参数。它的参数通过寄存器传递，除系统调用调用的函数（例如，sys_write）之外，还需指定被调用的函数本身。
2. 被调用的函数本身的代号，放在eax中，其他参数依次放到这些寄存器中：ebx（第一个参数）、ecx（第二个参数）、edx（第三个参数）、esi（第四个参数）、edi（第五个参数）。

> 看看吧。只要投入时间学起来，终究是比不学习要有点收获。
>
> 这个知识点，一点都不难。可它是学习其他知识的基础。不先学这个知识点，当我遇到建立在这个知识点基础上的知识时，就会感受到它的难了。就像写一个英语句子好像不难，但不会写英语单词的时候，写一个英语句子会很难。

这段程序有两个节（section），一个用来存放数据，一个用来存放代码。

程序的入口是`_start`，这是固定的。必须使用`global`导出`_start`，才能在使用`ld`链接时被找到。

## Linux下混合使用C和汇编

汇编。foo.asm

```assembly
extern choose

[section .data]
num1	dd	3
num2	dd	5

[section .text]
global	_start
global	my_print

_start:
	push	dword	num1
	push	dword num2
	call	choose
	add	esp, 8
	
	mov eax,	1				; sys_exit
	mov ebx, 0
	int 0x80
	

my_print:
	mov eax, 	4				; sys_write
	mov ebx,	1
	mov ecx,	[esp + 4]	; msg
	mov edx,	[esp + 8]		; len
	int 0x80
	ret
	
```



C代码。bar.c

```c
void my_print(char *msg, int len);

int choose(int a, int b)
{
		if(a >= b){
      my_printf("The 1thd one\n", 13);
    }else{
      my_printf("The 2thd one\n", 13)
    }
  
  	return 0;
}
```



编译

```shell
nasm -o foo.o foo.asm
gcc -c -o bar.o bar.c -m32
ld -s foo.o bar.o foo -m elf_i386
```



达到了这个标准，这块知识，就算掌握了，也够用了。唯一欠缺的是，把编译命令写到Makefile中。

## 好资料

The Linux Information Project

http://www.linfo.org/index.html

