# pmtest5c

## 代码

```assembly
; ==========================================
; pmtest5c.asm
; 编译方法：nasm pmtest5c.asm -o pmtest5c.com
; ==========================================

%include	"pm.inc"	; 常量, 宏, 以及一些说明

org	0100h
	jmp	LABEL_BEGIN

[SECTION .gdt]
; GDT
;                           段基址,       段界限     , 属性
LABEL_GDT:             Descriptor 0,                0, 0         ; 空描述符
LABEL_DESC_NORMAL:     Descriptor 0,           0ffffh, DA_DRW    ; Normal 描述符
LABEL_DESC_CODE32:     Descriptor 0,   SegCode32Len-1, DA_C+DA_32; 非一致代码段,32
LABEL_DESC_CODE16:     Descriptor 0,           0ffffh, DA_C      ; 非一致代码段,16
LABEL_DESC_CODE_DEST:  Descriptor 0, SegCodeDestLen-1, DA_C+DA_32; 非一致代码段,32
LABEL_DESC_CODE_RING3: Descriptor 0,SegCodeRing3Len-1, DA_C+DA_32+DA_DPL3
LABEL_DESC_DATA:       Descriptor 0,        DataLen-1, DA_DRW    ; Data	
LABEL_DESC_STACK:      Descriptor 0,       TopOfStack, DA_DRWA+DA_32;Stack, 32 位
LABEL_DESC_STACK3:     Descriptor 0,      TopOfStack3, DA_DRWA+DA_32+DA_DPL3
LABEL_DESC_LDT:        Descriptor 0,         LDTLen-1, DA_LDT    ; LDT
LABEL_DESC_TSS:        Descriptor 0,          TSSLen-1, DA_386TSS
LABEL_DESC_VIDEO:      Descriptor 0B8000h,     0ffffh, DA_DRW+DA_DPL3

; 门                               目标选择子,偏移,DCount, 属性
LABEL_CALL_GATE_TEST: Gate SelectorCodeDest,   0,     0, DA_386CGate+DA_DPL3
; GDT 结束

GdtLen		equ	$ - LABEL_GDT	; GDT长度
GdtPtr		dw	GdtLen - 1	; GDT界限
		dd	0		; GDT基地址

; GDT 选择子
SelectorNormal		equ	LABEL_DESC_NORMAL	- LABEL_GDT
SelectorCode32		equ	LABEL_DESC_CODE32	- LABEL_GDT
SelectorCode16		equ	LABEL_DESC_CODE16	- LABEL_GDT
SelectorCodeDest	equ	LABEL_DESC_CODE_DEST	- LABEL_GDT
SelectorCodeRing3	equ	LABEL_DESC_CODE_RING3	- LABEL_GDT + SA_RPL3
SelectorData		equ	LABEL_DESC_DATA		- LABEL_GDT
SelectorStack		equ	LABEL_DESC_STACK	- LABEL_GDT
SelectorStack3		equ	LABEL_DESC_STACK3	- LABEL_GDT + SA_RPL3
SelectorLDT		equ	LABEL_DESC_LDT		- LABEL_GDT
SelectorTSS		equ	LABEL_DESC_TSS		- LABEL_GDT
SelectorVideo		equ	LABEL_DESC_VIDEO	- LABEL_GDT

SelectorCallGateTest	equ	LABEL_CALL_GATE_TEST	- LABEL_GDT + SA_RPL3
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

; 堆栈段ring3
[SECTION .s3]
ALIGN	32
[BITS	32]
LABEL_STACK3:
	times 512 db 0
TopOfStack3	equ	$ - LABEL_STACK3 - 1
; END of [SECTION .s3]

; TSS
[SECTION .tss]
ALIGN	32
[BITS	32]
LABEL_TSS:
		DD	0			; Back
		DD	TopOfStack		; 0 级堆栈
		DD	SelectorStack		; 
		DD	0			; 1 级堆栈
		DD	0			; 
		DD	0			; 2 级堆栈
		DD	0			; 
		DD	0			; CR3
		DD	0			; EIP
		DD	0			; EFLAGS
		DD	0			; EAX
		DD	0			; ECX
		DD	0			; EDX
		DD	0			; EBX
		DD	0			; ESP
		DD	0			; EBP
		DD	0			; ESI
		DD	0			; EDI
		DD	0			; ES
		DD	0			; CS
		DD	0			; SS
		DD	0			; DS
		DD	0			; FS
		DD	0			; GS
		DD	0			; LDT
		DW	0			; 调试陷阱标志
		DW	$ - LABEL_TSS + 2	; I/O位图基址
		DB	0ffh			; I/O位图结束标志
TSSLen		equ	$ - LABEL_TSS


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

	; 初始化测试调用门的代码段描述符
	xor	eax, eax
	mov	ax, cs
	shl	eax, 4
	add	eax, LABEL_SEG_CODE_DEST
	mov	word [LABEL_DESC_CODE_DEST + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_CODE_DEST + 4], al
	mov	byte [LABEL_DESC_CODE_DEST + 7], ah

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

	; 初始化堆栈段描述符(Ring3)
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_STACK3
	mov	word [LABEL_DESC_STACK3 + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_STACK3 + 4], al
	mov	byte [LABEL_DESC_STACK3 + 7], ah

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

	; 初始化Ring3描述符
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_CODE_RING3
	mov	word [LABEL_DESC_CODE_RING3 + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_CODE_RING3 + 4], al
	mov	byte [LABEL_DESC_CODE_RING3 + 7], ah

	; 初始化 TSS 描述符
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_TSS
	mov	word [LABEL_DESC_TSS + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_TSS + 4], al
	mov	byte [LABEL_DESC_TSS + 7], ah

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

	mov	ax, SelectorTSS
	ltr	ax

	push	SelectorStack3
	push	TopOfStack3
	push	SelectorCodeRing3
	push	0
	retf

	ud2	; should never arrive here

	; 测试调用门（无特权级变换），将打印字母 'C'
	call	SelectorCallGateTest:0
	;call	SelectorCodeDest:0

	; Load LDT
	mov	ax, SelectorLDT
	lldt	ax

	jmp	SelectorLDTCodeA:0	; 跳入局部任务，将打印字母 'L'。

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


[SECTION .sdest]; 调用门目标段
[BITS	32]

LABEL_SEG_CODE_DEST:
	mov	ax, SelectorVideo
	mov	gs, ax			; 视频段选择子(目的)

	mov	edi, (80 * 12 + 0) * 2	; 屏幕第 12 行, 第 0 列。
	mov	ah, 0Ch			; 0000: 黑底    1100: 红字
	mov	al, 'C'
	mov	[gs:edi], ax

	retf

SegCodeDestLen	equ	$ - LABEL_SEG_CODE_DEST
; END of [SECTION .sdest]


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
LABEL_LDT:
;                                         段基址       段界限     ,   属性
LABEL_LDT_DESC_CODEA:	Descriptor	       0,     CodeALen - 1,   DA_C + DA_32	; Code, 32 位

LDTLen		equ	$ - LABEL_LDT

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

	mov	edi, (80 * 13 + 0) * 2	; 屏幕第 13 行, 第 0 列。
	mov	ah, 0Ch			; 0000: 黑底    1100: 红字
	mov	al, 'L'
	mov	[gs:edi], ax

	; 准备经由16位代码段跳回实模式
	jmp	SelectorCode16:0
CodeALen	equ	$ - LABEL_CODE_A
; END of [SECTION .la]

; CodeRing3
[SECTION .ring3]
ALIGN	32
[BITS	32]
LABEL_CODE_RING3:
	mov	ax, SelectorVideo
	mov	gs, ax

	mov	edi, (80 * 14 + 0) * 2
	mov	ah, 0Ch
	mov	al, '3'
	mov	[gs:edi], ax

	call	SelectorCallGateTest:0

	jmp	$
SegCodeRing3Len	equ	$ - LABEL_CODE_RING3
; END of [SECTION .ring3]
```



完全不理解这段代码。究竟是否会执行ud2后面的指令？



## 汇编

### ud2

UD2是一种让CPU产生invalid opcode exception的软件指令. 内核发现CPU出现这个异常, 会立即停止运行。

### 汇编指令速查

| 指令       | 功能               |
| ---------- | ------------------ |
| AAA        | 调整加             |
| AAD        | 调整除             |
| AAM        | 调整乘             |
| AAS        | 调整减             |
| ADC        | 进位加             |
| ADD        | 加                 |
| AND        | 与                 |
| ARPL       | 调整优先级         |
| BOUND      | 检查数组           |
| BSF        | 位右扫描           |
| BSR        | 位左扫描           |
| BSWAP      | 交换字节           |
| BT         | 位测试             |
| BTC        | 位测试求反         |
| BTR        | 位测试清零         |
| BTS        | 位测试置一         |
| CALL       | 过程调用           |
| CBW        | 转换字节           |
| CDQ        | 转换双字           |
| CLC        | 进位清零           |
| CLD        | 方向清零           |
| CLI        | 中断清零           |
| CLTS       | 任务清除           |
| CMC        | 进位求反           |
| CMOVA      | 高于传送           |
| CMOVB      | 低于传送           |
| CMOVE      | 相等传送           |
| CMOVG      | 大于传送           |
| CMOVL      | 小于传送           |
| CMOVNA     | 不高于传送         |
| CMOVNB     | 不低于传送         |
| CMOVNE     | 不等传送           |
| CMOVNG     | 不大于传送         |
| CMOVNL     | 不小于传送         |
| CMOVNO     | 不溢出传送         |
| CMOVNP     | 非奇偶传送         |
| CMOVNS     | 非负传送           |
| CMOVO      | 溢出传送           |
| CMOVP      | 奇偶传送           |
| CMOVS      | 负号传送           |
| CMP        | 比较               |
| CMPSB      | 比较字节串         |
| CMPSD      | 比较双字串         |
| CMPSW      | 比较字串           |
| CMPXCHG    | 比较交换           |
| CMPXCHG486 | 比较交换486        |
| CMPXCHG8B  | 比较交换8字节      |
| CPUID      | CPU标识            |
| CWD        | 转换字             |
| CWDE       | 扩展字             |
| DAA        | 调整加十           |
| DAS        | 调整减十           |
| DEC        | 减一               |
| DIV        | 除                 |
| ENTER      | 建立堆栈帧         |
| HLT        | 停                 |
| IDIV       | 符号整除           |
| IMUL       | 符号乘法           |
| IN         | 端口输入           |
| INC        | 加一               |
| INSB       | 端口输入字节串     |
| INSD       | 端口输入双字串     |
| INSW       | 端口输入字串       |
| **JA**     | **高于跳转**       |
| **JB**     | **低于跳转**       |
| **JBE**    | **不高于跳转**     |
| **JCXZ**   | **计数一六零跳转** |
| **JE**     | **相等跳转**       |
| **JECXZ**  | **计数三二零跳转** |
| **JG**     | **大于跳转**       |
| **JL**     | **小于跳转**       |
| **JMP**    | **跳转**           |
| **JMPE**   | **跳转扩展**       |
| **JNB**    | **不低于跳转**     |
| **JNE**    | **不等跳转**       |
| **JNG**    | **不大于跳转**     |
| **JNL**    | **不小于跳转**     |
| **JNO**    | **不溢出跳转**     |
| **JNP**    | **非奇偶跳转**     |
| **JNS**    | **非负跳转**       |
| **JO**     | **溢出跳转**       |
| **JP**     | **奇偶跳转**       |
| **JS**     | **负号跳转**       |
| LAHF       | 加载标志低八       |
| LAR        | 加载访问权限       |
| LDS        | 加载数据段         |
| LEA        | 加载有效地址       |
| LEAVE      | 清除过程堆栈       |
| LES        | 加载附加段         |
| LFS        | 加载标志段         |
| LGDT       | 加载全局描述符     |
| LGS        | 加载全局段         |
| LIDT       | 加载中断描述符     |
| LMSW       | 加载状态字         |
| LOADALL    | 加载所有           |
| LOADALL286 | 加载所有286        |
| LOCK       | 锁                 |
| LODSB      | 加载源变址字节串   |
| LODSD      | 加载源变址双字串   |
| LODSW      | 加载源变址字串     |
| LOOP       | 计数循环           |
| LOOPE      | 相等循环           |
| LOOPNE     | 不等循环           |
| LOOPNZ     | 非零循环           |
| LOOPZ      | 为零循环           |
| LSL        | 加载段界限         |
| LSS        | 加载堆栈段         |
| LTR        | 加载任务           |
| MONITOR    | 监视               |
| MOV        | 传送               |
| MOVSB      | 传送字节串         |
| MOVSD      | 传送双字串         |
| MOVSW      | 传送字串           |
| MOVSX      | 符号传送           |
| MOVZX      | 零传送             |
| MUL        | 乘                 |
| MWAIT      |                    |
| NEG        | 求补               |
| NOP        | 空                 |
| NOT        | 非                 |
| OR         | 或                 |
| OUT        | 端口输出           |
| OUTSB      | 端口输出字节串     |
| OUTSD      | 端口输出双字串     |
| OUTSW      | 端口输出字串       |
| POP        | 出栈               |
| POPA       | 全部出栈           |
| POPF       | 标志出栈           |
| PUSH       | 压栈               |
| PUSHA      | 全部压栈           |
| PUSHF      | 标志压栈           |
| RCL        | 进位循环左移       |
| RCR        | 进位循环右移       |
| RDMSR      | 读专用模式         |
| RDPMC      | 读执行监视计数     |
| RDSHR      |                    |
| RDTSC      | 读时间戳计数       |
| REP        | 重复               |
| REPE       | 相等重复           |
| REPNE      | 不等重复           |
| RET        | 过程返回           |
| RETF       | 远过程返回         |
| RETN       | 近过程返回         |
| ROL        | 循环左移           |
| ROR        | 循环右移           |
| RSM        | 恢复系统管理       |
| SAHF       | 恢复标志低八       |
| SAL        | 算术左移           |
| SALC       |                    |
| SAR        | 算术右移           |
| SBB        | 借位减             |
| SCASB      | 扫描字节串         |
| SCASD      | 扫描双字串         |
| SCASW      | 扫描字串           |
| SETA       | 高于置位           |
| SETB       | 低于置位           |
| SETE       | 相等置位           |
| SETG       | 大于置位           |
| SETL       | 小于置位           |
| SETNA      | 不高于置位         |
| SETNB      | 不低于置位         |
| SETNE      | 不等置位           |
| SETNG      | 不大于置位         |
| SETNL      | 不小于置位         |
| SETNO      | 不溢出置位         |
| SETNP      | 非奇偶置位         |
| SETNS      | 非负置位           |
| SETO       | 溢出置位           |
| SETP       | 奇偶置位           |
| SETS       | 负号置位           |
| SGDT       | 保存全局描述符     |
| SHL        | 逻辑左移           |
| SHLD       | 双精度左移         |
| SHR        | 逻辑右移           |
| SHRD       | 双精度右移         |
| SIDT       | 保存中断描述符     |
| SLDT       | 保存局部描述符     |
| SMI        |                    |
| SMINT      |                    |
| SMINTOLD   |                    |
| SMSW       | 保存状态字         |
| STC        | 进位设置           |
| STD        | 方向设置           |
| STI        | 中断设置           |
| STOSB      | 保存字节串         |
| STOSD      | 保存双字串         |
| STOSW      | 保存字串           |
| STR        | 保存任务           |
| SUB        | 减                 |
| SYSCALL    | 系统调用           |
| SYSENTER   | 系统进入           |
| SYSEXIT    | 系统退出           |
| SYSRET     | 系统返回           |
| TEST       | 数测试             |
| UD0        | 未定义指令0        |
| UD1        | 未定义指令1        |
| UD2        | 未定义指令2        |
| UMOV       |                    |
| VERW       | 校验写             |
| WAIT       | 等                 |
| WBINVD     | 回写无效高速缓存   |
| WRMSR      | 写专用模式         |
| WRSHR      |                    |
| XADD       | 交换加             |
| XBTS       |                    |
| XCHG       | 交换               |
| XLAT       | 换码               |
| XOR        | 异或               |
| XSTORE     |                    |