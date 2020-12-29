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

## 书本笔记

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

### 权限检查规则

![image-20201229194450054](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-cao-zuo-xi-tong/image-20201229194450054.png)

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

### b

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

### c

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

