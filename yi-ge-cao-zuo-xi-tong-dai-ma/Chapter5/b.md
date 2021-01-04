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



