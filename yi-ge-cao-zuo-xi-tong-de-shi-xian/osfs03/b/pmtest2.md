# pmtest2

## 代码

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
LABEL_DESC_TEST:   Descriptor 0500000h,     0ffffh, DA_DRW
LABEL_DESC_VIDEO:  Descriptor  0B8000h,     0ffffh, DA_DRW    ; 显存首地址。唯一需要留意的描述符。
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
ALIGN	32	; 作用是什么？
[BITS	32]
LABEL_DATA:
SPValueInRealMode	dw	0
; 字符串
PMMessage:		db	"In Protect Mode now. ^-^", 0	; 在保护模式中显示。这种语法，很陌生。
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
	times 512 db 0		; 初始化堆栈，512字节，每个字节都设置为0。

TopOfStack	equ	$ - LABEL_STACK - 1;	栈顶是当前位置减去栈开始的位置再减去1。稍微有点不理解，就把它当规则吧。桌子叫桌子，我需要知道为啥桌子叫桌子吗？减去1是因为，栈顶是最后一个元素的开始位置而不是结束位置。

; END of [SECTION .gs]


[SECTION .s16]
[BITS	16]
LABEL_BEGIN:
	mov	ax, cs
	mov	ds, ax
	mov	es, ax
	mov	ss, ax
	mov	sp, 0100h

	; LABEL_GO_BACK_TO_REAL 的第24位~39位，设置为ax。理解不了。
	mov	[LABEL_GO_BACK_TO_REAL+3], ax
	; 将SPValueInRealMode处内存设置为sp。
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
	xor	esi, esi
	xor	edi, edi
	mov	esi, OffsetPMMessage	; 源数据偏移
	mov	edi, (80 * 10 + 0) * 2	; 目的数据偏移。屏幕第 10 行, 第 0 列。
	; cld的作用是什么？将df设置为0，di会递增，增幅为1。
	cld
.1:
	; 把[ds:si]指向的内存单元中的数据复制到累加器(eax、ax、al、ah)中，并且si自增1
	lodsb
	; 执行 al and al，当al为0时，结果是0，zf是1，jz会跳转到.2
	test	al, al
	; 符合什么条件跳转到.2？
	jz	.2
	mov	[gs:edi], ax
	add	edi, 2
	jmp	.1
.2:	; 显示完毕

	call	DispReturn

	call	TestRead
	call	TestWrite
	call	TestRead

	; 到此停止
	jmp	SelectorCode16:0

; ------------------------------------------------------------------------
TestRead:
	xor	esi, esi			; esi设置为0
	mov	ecx, 8				; ecx是循环的次数
.loop:
	mov	al, [es:esi]
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
	test	al, al
	jz	.2
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
	mov	dl, al
	shr	al, 4
	; 循环2次。为啥要循环2次？
	; 因为数字0会被显示为00，数字A会被显示为0A。
	mov	ecx, 2
.begin:
	; 只保留al的第0~4位，al不超过2的4次方，即al最大值是15。
	and	al, 01111b
	cmp	al, 9
	; zf = 0 &7 cf = 0 时跳转到.1，此时 al > 9
	ja	.1
	add	al, '0'
	jmp	.2
.1:
	; al 大于 0Ah 时，将 al 打印为A-F之间的字母
	sub	al, 0Ah
	add	al, 'A'
.2:
	mov	[gs:edi], ax
	; 一个字符占用2个字节，分别是属性和ascii值
	add	edi, 2
	
	; 还原al的值
	mov	al, dl
	loop	.begin
	add	edi, 2

	pop	edx
	pop	ecx

	ret
; DispAL 结束-------------------------------------------------------------


; ------------------------------------------------------------------------
; 看不懂。div和mul指令的网络资料，与这里的代码对不上号。先搁置，找权威资料再理解，
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

; 仍然不理解这句。以前好像理解了的。这个标签的第24~39位被修改了。为啥只修改这几位？
LABEL_GO_BACK_TO_REAL:
	jmp	0:LABEL_REAL_ENTRY	; 段地址会在程序开始处被设置成正确的值

Code16Len	equ	$ - LABEL_SEG_CODE16

; END of [SECTION .s16code]
```



## 汇编知识

### loadsb

串操作指令LODSB/LODSW是块装入指令，其具体操作是把SI指向的存储单元读入累加器,LODSB就读入AL,LODSW就读入AX中,然后SI自动增加或减小1或2.其常常是对数组或字符串中的元素逐个进行处理。

字符串操作指令的实质是对一片连续存储单元进行处理，这片存储单元是由隐含指针DS:SI或ES:DI来指定的。字符串操作指令可对内存单元按字节、字或双字进行处理，并能根据操作对象的字节数使变址寄存器SI(和DI)增减1、2或4。具体规定如下：

(1)、当DF=0时，变址寄存器SI(和DI)增加1、2或4；
(2)、当DF=1时，变址寄存器SI(和DI)减少1、2或4。

在后面各指令中，有关变址寄存器都按上述规定进行增减，不再一一说明。

1、取字符串数据指令(Load String Instruction)

从由指针DS:SI所指向的内存单元开始，取一个字节、字或双字进入AL、AX或EAX中，并根据标志位DF对寄存器SI作相应增减。该指令的执行不影响任何标志位。

| 指令的格式：LODS  地址表达式 LODSB/LODSW LODSD　　　　　　　　　　;80386+在指令LODS中，它会根据其地址表达式的属性来决定读取一个字节、字或双字。即：当该地址表达式的属性为字节、字或双字时，将从指针DS:SI处读一个字节到AL中，或读一个字到AX，或读一个双字到EAX中，与此同时，SI还将分别增减1，2或4。 |      |
| ------------------------------------------------------------ | ---- |
|                                                              |      |

其它字符串指令中的“地址表达式”作用与此类似，将不再说明。

2、置字符串数据指令(Store String Instruction)

| 该指令是把寄存器AL、AX或EAX中的值存于以指针ES:DI所指向内存单元为起始的一片存储单元里，并根据标志位DF对寄存器DI作相应增减。该指令不影响任何标志位。指令的格式：STOS 地址表达式 STOSB/STOSW STOSD　　　　　　;80386+ |      |
| ------------------------------------------------------------ | ---- |
|                                                              |      |

3、字符串传送指令(Move String Instruction)该指令是把指针DS:SI所指向的字节、字或双字传送给指针ES:DI所指向内存单元，并根据标志位DF对寄存器DI和SI作相应增减。该指令的执行不影响任何标志位。指令的格式：MOVS 地址表达式1, 地址表达式2 MOVSB/MOVSW MOVSD　　　　　　;80386+

### cld

cld相对应的指令是std，二者均是用来操作方向标志位DF（Direction Flag）。cld使DF 复位，即是让DF=0，std使DF置位，即DF=1.这两个指令用于串操作指令中。通过执行cld或std指令可以控制方向标志DF，决定内存地址是增大（DF=0，向高地址增加）还是减小（DF=1，向地地址减小）。

### psw

![img](https://images0.cnblogs.com/blog/495899/201307/16213356-9ae1778981e244a9a686a17dbc73ac1a.jpg)



辅助记忆的C代码。这段代码，与上图矛盾，我觉得代码是错误的。

```c
void main(){
int a=3,b=5;
    if (a!=b) //je
        if (a==b) //jnz
            if (a<=b) //jg
                if (a<b) //jge
                    if (a>=b) //jl
                        if (a>b)//jle
                        {
                            printf("do if");
                        }
}
return 0;
```



标志寄存器PSW(程序状态字寄存器PSW)

 标志寄存器PSW是一个16位的寄存器。它反映了CPU运算的状态特征并且存放某些控制标志。8086使用了16位中的9位，包括6个状态标志位和3个控制标志位。

CF(进位标志位)：当执行一个加法（减法）运算时，最高位产生进位（或借位）时，CF为1，否则为0。
ZF零标志位：若当前的运算结果为零，则ZF为1，否则为0。
SF符号标志位：该标志位与运算结果的最高位相同。即运算结果为负，则SF为1，否则为0。
OF溢出标志位：若运算结果超出机器能够表示的范围称为溢出，此时OF为1，否则为0。判断是否溢出的方法是：进行二进制运算时，最高位的进位值与次高位的进位值进行异或运算，若运算结果为1则表示溢出OF=1，否则OF=0 （？）

PF奇偶标志：当运算结果的最低16位中含1的个数为偶数则PF=1否则PF=0
AF辅助进位标志：一个加法（减法）运算结果的低4位向高4位有进位（或借位）时则AF=1否则AF=0 

  另外还有三个控制标志位用来控制CPU的操作，可以由程序进行置位和复位。
TF跟踪标志：该标志位为方便程序调试而设置。若TF=1，8086/8088CPU处于单步工作方式，即在每条指令执行结束后，产生中断。
IF中断标志位：该标志位用来控制CPU是否响应可屏蔽中断。若IF=1则允许中断，否则禁止中断。
DF方向标志：该标志位用来控制串处理指令的处理方向。若DF=1则串处理过程中地址自动递减，否则自动递增。

### test

TEST 指令在两个操作数的对应位之间进行 AND 操作，并根据运算结果设置符号标志位、零标志位和奇偶标志位。

不改变操作数。

不是很理解。

### cmp

CMP（比较）指令执行从目的操作数中减去源操作数的隐含减法操作，并且不修改任何操作数：

CMP destination,source

#### 标志位

当实际的减法发生时，CMP 指令按照计算结果修改溢出、符号、零、进位、辅助进位和奇偶标志位。

如果比较的是两个无符号数，则零标志位和进位标志位表示的两个操作数之间的关系如右表所示：



| CMP结果               | ZF   | CF   |
| --------------------- | ---- | ---- |
| 目的操作数 < 源操作数 | 0    | 1    |
| 目的操作数 > 源操作数 | 0    | 0    |
| 目的操作数 = 源操作数 | 1    | 0    |


如果比较的是两个有符号数，则符号标志位、零标志位和溢出标志位表示的两个操作数之间的关系如右表所示：



| CMP结果               | 标志位  |
| --------------------- | ------- |
| 目的操作数 < 源操作数 | SF ≠ OF |
| 目的操作数 > 源操作数 | SF=OF   |
| 目的操作数 = 源操作数 | ZF=1    |


CMP 指令是创建条件逻辑结构的重要工具。当在条件跳转指令中使用 CMP 时，汇编语言的执行结果就和 IF 语句一样。

下面用三段代码来说明标志位是如何受到 CMP 影响的。设 AX=5，并与 10 进行比较，则进位标志位将置 1，原因是（5-10）需要借位：

mov ax, 5
cmp ax,10   ; ZF = 0 and CF = 1

1000 与 1000 比较会将零标志位置 1，因为目标操作数减去源操作数等于 0：

mov ax,1000
mov cx,1000
cmp cx, ax    ;ZF = 1 and CF = 0

105 与 0 进行比较会清除零和进位标志位，因为（105-0）的结果是一个非零的正整数。

mov si,105
cmp si, 0    ; ZF = 0 and CF = 0