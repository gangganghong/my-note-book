# pmtest1.asm

```assembly
; ==========================================
; pmtest1.asm
; 编译方法：nasm pmtest1.asm -o pmtest1.bin
; ==========================================

%include	"pm.inc"	; 常量, 宏, 以及一些说明

; 第一条指令的内存地址是07c00h
org	07c00h
; 跳转到LABEL_BEGIN，这是执行的第一条指令
	jmp	LABEL_BEGIN

; gdt 区域
[SECTION .gdt]
; GDT
;                              段基址,       段界限     , 属性
LABEL_GDT:	   Descriptor       0,                0, 0           ; 空描述符
LABEL_DESC_CODE32: Descriptor       0, SegCode32Len - 1, DA_C + DA_32; 非一致代码段
LABEL_DESC_VIDEO:  Descriptor 0B8000h,           0ffffh, DA_DRW	     ; 显存首地址
; GDT 结束

; 当前位置$减去第一个gdt的位置LABEL_GDT，结果是GDT的长度。
GdtLen		equ	$ - LABEL_GDT	; GDT长度
; GdtPtr的第0~7位是GdtLen - 1，第8~23位是0。
; GDT的长度减去1，就是GDT界限了。
GdtPtr		dw	GdtLen - 1	; GDT界限
		dd	0		; GDT基地址

; GDT 选择子。选择子是GDT相对于第一个GDT（空）的偏移量，单位是8个字节，即64位。
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
	; 上面的语句，将ds、es、ss、cs初始化为相同的值。
	; 那么，cs的值是什么呢？它的值又是从何而来呢？为何第一句不是 mov ax, ds呢？
	; 栈顶地址是0100h。为何设置成这个地址？
	mov	sp, 0100h

	; 初始化 32 位代码段描述符
	; 设置eax为0。xor 1,1 结果是0。
	xor	eax, eax
	; 代码段左移动四位 + 偏移量，是内存地址。
	; 初始化全局描述符。又花了很长时间才理解。
	; 代码段左移动四位 + 偏移量，结果是段基址还是内存地址？段基址也是一个内存地址。
	; LABEL_SEG_CODE32 是偏移量。
	mov	ax, cs
	shl	eax, 4
	; LABEL_SEG_CODE32 是段偏移量
	add	eax, LABEL_SEG_CODE32
	; ax 是内存地址的第0~15位。LABEL_DESC_CODE32 + 2 是描述符的第16位。
	; word是一个字，16位，描述符的第16~31位的值是ax，即这个段的内存地址。
	mov	word [LABEL_DESC_CODE32 + 2], ax
	; 右移动16位，是内存地址的第16位~31位
	shr	eax, 16
	; 描述符的第32~39位，是段基址的第16位~23位。
	mov	byte [LABEL_DESC_CODE32 + 4], al
	; 描述符的第56~63位，是段基址的第24位~31位。
	mov	byte [LABEL_DESC_CODE32 + 7], ah

	; 为加载 GDTR 作准备
	; eax 初始化为0。
	xor	eax, eax
	; ax 赋值为数据段
	mov	ax, ds
	; eax 左移动4位，
	shl	eax, 4
	; LABEL_GDT 也是一个内存地址，内存地址的计算方法都是：段基址 << 4 + 偏移量
	add	eax, LABEL_GDT		; eax <- gdt 基地址
	; GdtPtr 的第16~47位是 gdt 基址
	mov	dword [GdtPtr + 2], eax	; [GdtPtr + 2] <- gdt 基地址

	; 加载 GDTR
	lgdt	[GdtPtr]

	; 关中断
	cli

	; 打开地址线A20
	in	al, 92h
	; 将端口92h的第2位（低2位）设置为1
	or	al, 00000010b
	out	92h, al

	; 准备切换到保护模式
	; 将cr0设置为1
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
	
	; 打印字符串的另外一种方式，直接写数据到显存
	mov	edi, (80 * 11 + 79) * 2	; 屏幕第 11 行, 第 79 列。
	mov	ah, 0Ch			; 0000: 黑底    1100: 红字
	mov	al, 'P'
	mov	[gs:edi], ax

	; 到此停止
	jmp	$

; 不要这句可以吗？
SegCode32Len	equ	$ - LABEL_SEG_CODE32
; END of [SECTION .s32]
```

