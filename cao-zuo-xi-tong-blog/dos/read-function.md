# 汇编函数阅读笔记

## memset

### 原型

```c
void memset(void* p_dst, char ch, int size)
```

这是`memset`的函数原型，在C语言中使用这个函数时，需按这个原型传参。

`memset`的功能是：用`size`个`char`类型的数据填充初始内存地址是`p_dst`的这片内存空间。

### 代码

```assembly
global	memset

memset:
	push	ebp
	mov	ebp, esp

	push	esi
	push	edi
	push	ecx

	mov	edi, [ebp + 8]	; Destination
	mov	edx, [ebp + 12]	; Char to be putted
	mov	ecx, [ebp + 16]	; Counter
.1:
	cmp	ecx, 0		; 判断计数器
	jz	.2		; 计数器为零时跳出

	mov	byte [edi], dl		; ┓
	inc	edi			; ┛

	dec	ecx		; 计数器减一
	jmp	.1		; 循环
.2:

	pop	ecx
	pop	edi
	pop	esi
	mov	esp, ebp
	pop	ebp

	ret			; 函数结束，返回
```

### 解读

#### 函数模板

`nasm`汇编写函数的模板是：

```assembly
; 函数名
funcName:
	; 被修改的寄存器都要事先存储到堆栈中，所以,ebp、eax、ebx、ecx都要入栈
	push	ebp
	mov		ebp,	esp
	
	push	eax
	push	ebx
	push	ecx
	
	; 调用funcName时，参数按照从右到左依次入栈
	; ebp + 0 是 eip，ebp + 4 是esp
	mov	eax,	[ebp + 16]	; 第三个参数
	mov	ebx,	[ebp + 12]	; 第二个参数
	mov	ecx,	[ebp + 8]		; 第一个参数
	
	; some code
	; some code
	
	; 在函数末尾通过出栈还原被修改过的寄存器中的值，出栈顺序和签名的入栈顺序相同
	pop	ecx
	pop	ebx
	pop	eax
	pop	ebp
	
	; 函数末尾必须用这个指令结尾，出栈esp和eip。
	ret
```

#### 使用

要在其他文件中使用这个函数，需在本文件使用`global	memset`将此函数导出。

#### 其他

```assembly
.1:
	cmp	ecx, 0		; 判断计数器
	jz	.2		; 计数器为零时跳出

	mov	byte [edi], dl		; ┓
	inc	edi			; ┛

	dec	ecx		; 计数器减一
	jmp	.1		; 循环
```

`dl`是第二个参数`char ch`。`char`是8个字节，因此只需要使用寄存器`dl`。

`mov	byte [edi], dl`把`ch`填充到`es:edi`内存空间。

```assembly
.1:
	;some code
	;some code
	jmp .1
```

用`jmp`实现循环指令。能不用`loop`指令就不用。`loop`指令必须与`ecx`配合使用，非常容易出错。

## out_byte

```assembly
; 函数名称是 out_byte
out_byte:
	; esp 是 eip
	mov	edx, [esp + 4]		; port，第一个参数
	mov	al, [esp + 4 + 4]	; value，第二个参数
	; 把al写入dx端口
	out	dx, al
	nop	; 一点延迟
	nop
	; 堆栈中的eip出栈
	ret
```

## in_byte

```assembly
; 函数名称是 in_byte
in_byte:
	mov	edx, [esp + 4]		; port，第一个参数
	xor	eax, eax	; 设置eax的值是0。异或运算，不相等结果是1，相等结果是0。
	in	al, dx		; 把dx端口的值写入dl
	nop	; 一点延迟
	nop
	ret
```

## disp_str

### 代码

```assembly
disp_str:
	push	ebp
	mov	ebp, esp

	mov	esi, [ebp + 8]	; pszInfo
	mov	edi, [disp_pos]
	mov	ah, 0Fh
.1:
	lodsb
	test	al, al
	jz	.2
	cmp	al, 0Ah	; 是回车吗?
	jnz	.3
	push	eax
	mov	eax, edi
	mov	bl, 160
	div	bl
	and	eax, 0FFh
	inc	eax
	mov	bl, 160
	mul	bl
	mov	edi, eax
	pop	eax
	jmp	.1
.3:
	mov	[gs:edi], ax
	add	edi, 2
	jmp	.1

.2:
	mov	[disp_pos], edi

	pop	ebp
	ret
```

理解这个函数花了很多时间。原因是没有及时联想到读写显存的坐标知识。

### 流程

流程是：

1. 从参数`pszInfo`加载一个字节数据到`al`。
2. 测试`al`是否为空。
   1. 为空，函数结束。
   2. 非空
      1. 字符不是`回车`，打印字符，视频段偏移量自增2个字节，然后跳转到最外层流程1。
      2. 字符是回车，不打印这个字符，
         1. 计算出视频偏移量。计算方法是：
            1. 先计算出当前视频偏移量在第`N`行，计算公式是：`视频偏移量/160`。
            2. 要打印的新字符所在行数应该是：`N+1`。
            3. 将新行数转换成视频偏移量：`(N+1)*160`。
         2. 跳转到最外层流程1。

#### 显存坐标

`[gs:(80*1+0)*2]`，把字符写入第1行第1列。

`[gs:(80*2+1)*2]`，把字符写入第1行第2列。

### 指令

#### lodsb

`lodsb`，把`[ds:si]`处的数据读入`al`，`si`自动自增1。

#### test

```assembly
test	al, al
jz	.2
```

`al`为空时，跳转到`.2`。

#### cmp

```assembly
cmp	al, 0Ah	; 是回车吗?
jnz	.3
```

`al`的值不是`0Ah`时，跳转到`.3`。`0Ah`是回车键的ASCII码。

#### div

```assembly
mov	eax, edi
mov	bl, 160
div	bl
and	eax, 0FFh
```

`div`是除法。被除数在`eax`中，除数在`bl`中，商在`al`中，余数在`ah`中。

`and	eax, 0FFh`获取`al`中的值。

## disp_color_str

```assembly
disp_color_str:
	push	ebp
	mov	ebp, esp

	mov	esi, [ebp + 8]	; pszInfo
	mov	edi, [disp_pos]
	mov	ah, [ebp + 12]	; color
.1:
	lodsb
	test	al, al
	jz	.2
	cmp	al, 0Ah	; 是回车吗?
	jnz	.3
	push	eax
	mov	eax, edi
	mov	bl, 160
	div	bl
	and	eax, 0FFh
	inc	eax
	mov	bl, 160
	mul	bl
	mov	edi, eax
	pop	eax
	jmp	.1
.3:
	mov	[gs:edi], ax
	add	edi, 2
	jmp	.1

.2:
	mov	[disp_pos], edi

	pop	ebp
	ret
```

只在`disp_str`的基础上增加了一句`mov	ah, [ebp + 12]	; color`，设置打印字符串的颜色，其他部分与`disp_str`完全一致。

## restart

```assembly
restart:
	mov	esp, [p_proc_ready]
	lldt	[esp + P_LDT_SEL] 
	lea	eax, [esp + P_STACKTOP]
	mov	dword [tss + TSS3_S_SP0], eax

	pop	gs
	pop	fs
	pop	es
	pop	ds
	popad

	add	esp, 4

	iretd
```

这是构造好进程后，启动第一个进程使用的函数。

语法很好懂，难点在业务逻辑。

### 指令

#### lldt

`lldt	[esp + P_LDT_SEL] `，加载LDT。`[esp + P_LDT_SEL]`是LDT的选择子。

#### lea

假设，si = 1000h,ds = 50000h,（51000h)=1234h，那么

1. `lea ax, [ds:si]`执行后，ax的值是`1000h`。
2. `mov ax, [d:si]`执行后，ax的值是`1234h`。

`lea	eax, [esp + P_STACKTOP]`执行后，`eax`的值是一个偏移量，是内存地址，而不是内存`esp + P_STACKTOP`中的值。

#### popad

`popad`依次出栈`EDI,ESI,EBP,ESP,EBX,EDX,ECX,EAX`。

#### iretd

暂时没有找到权威资料。

### 业务逻辑

1. `[p_proc_ready]`指向进程表的初始位置。j
2. 初始位置正好是存储寄存器的结构regs。
3. `P_LDT_SEL`是regs的长度。`[esp + P_LDT_SEL] `跳过regs，指向进程表的第二个成员结构，是LDT的选择子。
4. 其实`[esp + P_LDT_SEL] `和`[esp + P_STACKTOP]`指向同一个内存单元。总之，这个时候，指向regs的栈顶。
5. `mov	dword [tss + TSS3_S_SP0], eax`，让`TSS`的`sp0`指向regs的栈顶。
6. 接下来，执行pop操作时，CPU挑选的是0特权级的`ss0`和`sp0`，所以，出栈的栈是进程表中的regs。