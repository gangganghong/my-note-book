# 函数阅读笔记

## itoa

```c
PUBLIC char * itoa(char * str, int num)/* 数字前面的 0 不被显示出来, 比如 0000B800 被显示成 B800 */
{
	char *	p = str;
	char	ch;
	int	i;
	int	flag = FALSE;

	*p++ = '0';
	*p++ = 'x';

	if(num == 0){
		*p++ = '0';
	}
	else{	
		for(i=28;i>=0;i-=4){
			ch = (num >> i) & 0xF;
			if(flag || (ch > 0)){
				flag = TRUE;
				ch += '0';
				if(ch > '9'){
					ch += 7;
				}
				*p++ = ch;
			}
		}
	}

	*p = 0;

	return str;
}
```

功能是，把整型数字转换成字符串类型的十六进制数，数字前面的0不显示。

流程是：

1. 存储结果的是`str`，要被转换的数字是`num`。
2. 先在存储结果的字符串前面加上`0x`。
3. 依次获取`num`的第31位~第28位、第27位~第24位、第23位~第20位、第19位~第16位、第15位~第12位、第11位~第8位、第7位~第4位、第3位~第0位。
4. 也就是说，将`num`的二进制分成8组（每组四位，刚好32位）。
5. 将每组的值`value`加上字符`0`，结果是`asciiCode`。
6. 比较`asciiCode`和字符`9`，按下面的流程对`asciiCode`进行转换，然后得到`asciiCode`的字符形式`ch`。
   1. 比字符`9`小。字符`0~~9`正好和整型`0~~9`的字符形式一一对应，不需要转换。
   2. 比字符`9`大，将asciiCode加7
      1. 大于整型`9`的整型数字`10、11、12、13、14、15`（将这批数字记作`nums`）对应的字符应该是`A、B、C、D、E、F`。
      2. `nums+'0'`的结果并不是`A、B、C、D、E、F`。要让`nums`与`A、B、C、D、E、F`一一对应，需要将asciiCode加7。
      3. 这么一个简单的逻辑，要用文字叙述清楚，很不容易。甚至上面这段文字叙述得就不清楚。
   3. 这么叙述吧。
      1. 整型数字`0~~9`与字符`0~~9`的一一对应的差值是字符`0`的ascii码。
      2. 整数数字`10~~15`与字符`A~~F`一一对应的差值是字符`0`的ascii码加7。
      3. 非要问是怎么得出上面两个结论的，回答是：举例、观察、归纳。
7. 将`ch`拼接到`str`。
8. 最后一句`*p = 0;`是否有必要，无法心算。比较容易纠结这种corner case。

## exception_handler

```c
/*======================================================================*
                            exception_handler
 *----------------------------------------------------------------------*
 异常处理
 *======================================================================*/
PUBLIC void exception_handler(int vec_no, int err_code, int eip, int cs, int eflags)
{
	int i;
	int text_color = 0x74; /* 灰底红字 */
	char err_description[][64] = {	"#DE Divide Error",
					"#DB RESERVED",
					"—  NMI Interrupt",
					"#BP Breakpoint",
					"#OF Overflow",
					"#BR BOUND Range Exceeded",
					"#UD Invalid Opcode (Undefined Opcode)",
					"#NM Device Not Available (No Math Coprocessor)",
					"#DF Double Fault",
					"    Coprocessor Segment Overrun (reserved)",
					"#TS Invalid TSS",
					"#NP Segment Not Present",
					"#SS Stack-Segment Fault",
					"#GP General Protection",
					"#PF Page Fault",
					"—  (Intel reserved. Do not use.)",
					"#MF x87 FPU Floating-Point Error (Math Fault)",
					"#AC Alignment Check",
					"#MC Machine Check",
					"#XF SIMD Floating-Point Exception"
				};

	/* 通过打印空格的方式清空屏幕的前五行，并把 disp_pos 清零 */
	disp_pos = 0;
	for(i=0;i<80*5;i++){
		disp_str(" ");
	}
	disp_pos = 0;

	disp_color_str("Exception! --> ", text_color);
	disp_color_str(err_description[vec_no], text_color);
	disp_color_str("\n\n", text_color);
	disp_color_str("EFLAGS:", text_color);
	disp_int(eflags);
	disp_color_str("CS:", text_color);
	disp_int(cs);
	disp_color_str("EIP:", text_color);
	disp_int(eip);

	if(err_code != 0xFFFFFFFF){
		disp_color_str("Error code:", text_color);
		disp_int(err_code);
	}
}
```

`int eip, int cs, int eflags`这三个参数是`call`入栈的。挺奇特的语法。

## init_descriptor

```c
PRIVATE void init_descriptor(DESCRIPTOR *p_desc,u32 base,u32 limit,u16 attribute)
{
	p_desc->limit_low	= limit & 0x0FFFF;
	p_desc->base_low	= base & 0x0FFFF;
	p_desc->base_mid	= (base >> 16) & 0x0FF;
	p_desc->attr1		= attribute & 0xFF;
	p_desc->limit_high_attr2= ((limit>>16) & 0x0F) | (attribute>>8) & 0xF0;
	p_desc->base_high	= (base >> 24) & 0x0FF;
}
```

初始化描述符。

`p_desc->limit_high_attr2= ((limit>>16) & 0x0F) | (attribute>>8) & 0xF0;`，`attribute`在高4位，`limit`在低4位。

## seg2phys

```c
/*======================================================================*
                           seg2phys
 *----------------------------------------------------------------------*
 由段名求绝对地址
 *======================================================================*/
PUBLIC u32 seg2phys(u16 seg)
{
	DESCRIPTOR* p_dest = &gdt[seg >> 3];
	return (p_dest->base_high<<24 | p_dest->base_mid<<16 | p_dest->base_low);
}
```

流程如下：

1. `seg`是选择子。选择子的高13位是描述符在GDT中的索引。
2. 描述符中存储着段的基地址，把高中低三部分用`|`拼接起来。

## init_idt_desc

```c
/*======================================================================*
                             init_idt_desc
 *----------------------------------------------------------------------*
 初始化 386 中断门
 *======================================================================*/
PUBLIC void init_idt_desc(unsigned char vector, u8 desc_type, int_handler handler, unsigned char privilege)
{
	GATE *	p_gate	= &idt[vector];
	u32	base	= (u32)handler;
	p_gate->offset_low	= base & 0xFFFF;
	p_gate->selector	= SELECTOR_KERNEL_CS;
	p_gate->dcount		= 0;
	p_gate->attr		= desc_type | (privilege << 5);
	p_gate->offset_high	= (base >> 16) & 0xFFFF;
}
```

`p_gate->attr		= desc_type | (privilege << 5);`，很费解。

![image-20210306135703856](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/dos/code/image-20210306135703856.png)

`desc_type`是图中的`A`，`privilege`是图中的`B`。

费解的原因是，误以为`privilege`是纯`DPL`，``desc_type`是纯`TYPE`。

## vir2phys

```c
/* 宏 */
/* 线性地址 → 物理地址 */
#define vir2phys(seg_base, vir)	(u32)(((u32)seg_base) + (u32)(vir))
```

把线性地址转换成物理地址，这是保护模式下的计算公式。

`seg_base`是段基址。

## init_8259A

```c
/*======================================================================*
                            init_8259A
 *======================================================================*/
PUBLIC void init_8259A()
{
	out_byte(INT_M_CTL,	0x11);			// Master 8259, ICW1.
	out_byte(INT_S_CTL,	0x11);			// Slave  8259, ICW1.
	out_byte(INT_M_CTLMASK,	INT_VECTOR_IRQ0);	// Master 8259, ICW2. 设置 '主8259' 的中断入口地址为 0x20.
	out_byte(INT_S_CTLMASK,	INT_VECTOR_IRQ8);	// Slave  8259, ICW2. 设置 '从8259' 的中断入口地址为 0x28
	out_byte(INT_M_CTLMASK,	0x4);			// Master 8259, ICW3. IR2 对应 '从8259'.
	out_byte(INT_S_CTLMASK,	0x2);			// Slave  8259, ICW3. 对应 '主8259' 的 IR2.
	out_byte(INT_M_CTLMASK,	0x1);			// Master 8259, ICW4.
	out_byte(INT_S_CTLMASK,	0x1);			// Slave  8259, ICW4.

	out_byte(INT_M_CTLMASK,	0xFF);	// Master 8259, OCW1. 
	out_byte(INT_S_CTLMASK,	0xFF);	// Slave  8259, OCW1. 
}
```

没有记住，部分没有理解。

## init_prot

```c
/*======================================================================*
                            init_prot
 *----------------------------------------------------------------------*
 初始化 IDT
 *======================================================================*/
PUBLIC void init_prot()
{
	init_8259A();

	// 全部初始化成中断门(没有陷阱门)
	init_idt_desc(INT_VECTOR_DIVIDE,	DA_386IGate, divide_error,		PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_DEBUG,		DA_386IGate, single_step_exception,	PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_NMI,		DA_386IGate, nmi,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_BREAKPOINT,	DA_386IGate, breakpoint_exception,	PRIVILEGE_USER);
	init_idt_desc(INT_VECTOR_OVERFLOW,	DA_386IGate, overflow,			PRIVILEGE_USER);
	init_idt_desc(INT_VECTOR_BOUNDS,	DA_386IGate, bounds_check,		PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_INVAL_OP,	DA_386IGate, inval_opcode,		PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_COPROC_NOT,	DA_386IGate, copr_not_available,	PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_DOUBLE_FAULT,	DA_386IGate, double_fault,		PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_COPROC_SEG,	DA_386IGate, copr_seg_overrun,		PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_INVAL_TSS,	DA_386IGate, inval_tss,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_SEG_NOT,	DA_386IGate, segment_not_present,	PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_STACK_FAULT,	DA_386IGate, stack_exception,		PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_PROTECTION,	DA_386IGate, general_protection,	PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_PAGE_FAULT,	DA_386IGate, page_fault,		PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_COPROC_ERR,	DA_386IGate, copr_error,		PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_IRQ0 + 0,	DA_386IGate, hwint00,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_IRQ0 + 1,	DA_386IGate, hwint01,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_IRQ0 + 2,	DA_386IGate, hwint02,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_IRQ0 + 3,	DA_386IGate, hwint03,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_IRQ0 + 4,	DA_386IGate, hwint04,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_IRQ0 + 5,	DA_386IGate, hwint05,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_IRQ0 + 6,	DA_386IGate, hwint06,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_IRQ0 + 7,	DA_386IGate, hwint07,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_IRQ8 + 0,	DA_386IGate, hwint08,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_IRQ8 + 1,	DA_386IGate, hwint09,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_IRQ8 + 2,	DA_386IGate, hwint10,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_IRQ8 + 3,	DA_386IGate, hwint11,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_IRQ8 + 4,	DA_386IGate, hwint12,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_IRQ8 + 5,	DA_386IGate, hwint13,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_IRQ8 + 6,	DA_386IGate, hwint14,			PRIVILEGE_KRNL);
	init_idt_desc(INT_VECTOR_IRQ8 + 7,	DA_386IGate, hwint15,			PRIVILEGE_KRNL);

	/* 填充 GDT 中 TSS 这个描述符 */
	memset(&tss, 0, sizeof(tss));
	tss.ss0		= SELECTOR_KERNEL_DS;
	init_descriptor(&gdt[INDEX_TSS],
			vir2phys(seg2phys(SELECTOR_KERNEL_DS), &tss),
			sizeof(tss) - 1,
			DA_386TSS);
	tss.iobase	= sizeof(tss);	/* 没有I/O许可位图 */

	// 填充 GDT 中进程的 LDT 的描述符
	init_descriptor(&gdt[INDEX_LDT_FIRST],
			vir2phys(seg2phys(SELECTOR_KERNEL_DS), proc_table[0].ldts),
			LDT_SIZE * sizeof(DESCRIPTOR) - 1,
			DA_LDT);

}
```

`init_8259A();`，初始化`8259A`。

`init_idt_desc(INT_VECTOR_IRQ8 + 0,	DA_386IGate, hwint08,			PRIVILEGE_KRNL);`，`INT_VECTOR_IRQ8`是`0x20`。中断向量号是`IDT`的索引。

```c
/* 填充 GDT 中 TSS 这个描述符 */
	memset(&tss, 0, sizeof(tss));
	tss.ss0 = SELECTOR_KERNEL_DS;
	init_descriptor(&gdt[INDEX_TSS],
			vir2phys(seg2phys(SELECTOR_KERNEL_DS), &tss),
			sizeof(tss) - 1,
			DA_386TSS);
	tss.iobase = sizeof(tss); /* 没有I/O许可位图 */
```

### TSS

TSS也是一块内存空间，在GDT中会注册一个描述符。初始化一个描述符，需要：

1. 这块内存的初始地址（段基址）。
2. 这块内存的界限，等于这块内存的长度减去1。
3. 这块内存的属性。

```c
typedef struct s_tss {
	u32	backlink;
	u32	esp0;	/* stack pointer to use during interrupt */
	u32	ss0;	/*   "   segment  "  "    "        "     */
	u32	esp1;
	u32	ss1;
	u32	esp2;
	u32	ss2;
	u32	cr3;
	u32	eip;
	u32	flags;
	u32	eax;
	u32	ecx;
	u32	edx;
	u32	ebx;
	u32	esp;
	u32	ebp;
	u32	esi;
	u32	edi;
	u32	es;
	u32	cs;
	u32	ss;
	u32	ds;
	u32	fs;
	u32	gs;
	u32	ldt;
	u16	trap;
	u16	iobase;	/* I/O位图基址大于或等于TSS段界限，就表示没有I/O许可位图 */
}TSS;

/* GDT */
/* 描述符索引 */
#define	INDEX_DUMMY		0	/* \                         */
#define	INDEX_FLAT_C		1	/* | LOADER 里面已经确定了的 */
#define	INDEX_FLAT_RW		2	/* |                         */
#define	INDEX_VIDEO		3	/* /                         */
#define	INDEX_TSS		4
#define	INDEX_LDT_FIRST		5
/* 选择子 */
#define	SELECTOR_DUMMY		   0	/* \                         */
#define	SELECTOR_FLAT_C		0x08	/* | LOADER 里面已经确定了的 */
#define	SELECTOR_FLAT_RW	0x10	/* |                         */
#define	SELECTOR_VIDEO		(0x18+3)/* /<-- RPL=3                */
#define	SELECTOR_TSS		0x20	/* TSS                       */
#define SELECTOR_LDT_FIRST	0x28

#define	SELECTOR_KERNEL_CS	SELECTOR_FLAT_C
#define	SELECTOR_KERNEL_DS	SELECTOR_FLAT_RW
#define	SELECTOR_KERNEL_GS	SELECTOR_VIDEO

/* 每个任务有一个单独的 LDT, 每个 LDT 中的描述符个数: */
#define LDT_SIZE		2
```

`tss.iobase = sizeof(tss); /* 没有I/O许可位图 */`，把io位图的基地址设置为TSS的长度，等同于设置这个TSS没有IO位图。上面的说法有问题，我们的tss结构中本来就没有IO位图。

### LDT

```assembly
// 填充 GDT 中进程的 LDT 的描述符
	init_descriptor(&gdt[INDEX_LDT_FIRST],
			vir2phys(seg2phys(SELECTOR_KERNEL_DS), proc_table[0].ldts),
			LDT_SIZE * sizeof(DESCRIPTOR) - 1,
			DA_LDT);
```

没有特别之处，与初始化TSS相同。

## hwint00

```assembly
ALIGN	16
hwint00:		; Interrupt routine for irq 0 (the clock).
	sub	esp, 4
	pushad		; `.
	push	ds	;  |
	push	es	;  | 保存原寄存器值
	push	fs	;  |
	push	gs	; /
	mov	dx, ss
	mov	ds, dx
	mov	es, dx

	inc	byte [gs:0]		; 改变屏幕第 0 行, 第 0 列的字符

	mov	al, EOI			; `. reenable
	out	INT_M_CTL, al		; /  master 8259

	inc	dword [k_reenter]
	cmp	dword [k_reenter], 0
	jne	.re_enter
	
	mov	esp, StackTop		; 切到内核栈

	sti
	
	push	clock_int_msg
	call	disp_str
	add	esp, 4

;;; 	push	1
;;; 	call	delay
;;; 	add	esp, 4
	
	cli
	
	mov	esp, [p_proc_ready]	; 离开内核栈

	lea	eax, [esp + P_STACKTOP]
	mov	dword [tss + TSS3_S_SP0], eax

.re_enter:	; 如果(k_reenter != 0)，会跳转到这里
	dec	dword [k_reenter]
	pop	gs	; `.
	pop	fs	;  |
	pop	es	;  | 恢复原寄存器值
	pop	ds	;  |
	popad		; /
	add	esp, 4

	iretd
```

### 中断重入

一个中断发生并且处理还没有结束时，又产生一个中断。这就叫中断重入。

设置一个全局变量`flag`，初始值为-1。进入中断例程时，flag加1。中断处理结束时，flag减去1。进入中断处理前先检查flag的值，当flag的值不是0时，说明在一个中断还未处理结束的过程中又发生了一个中断，直接结束中断例程。

### 中断过程中的堆栈切换

`ring0`：特权级0；`ring1`：特权级1。

#### 从内核栈到进程栈

启动第一个进程前，设置好了TSS中的ss0（在`init_port`中设置）和sp0（在restart中设置）。

注意，不要先入为主地认为ss0和sp0应该有什么关系。ss0是ring0的堆栈选择子，sp0是ring0的堆栈栈顶。

时钟中断发生时，CPU自动从TSS中选择选择ss0和sp0。而sp0指向进程表中的regs。进程的堆栈存储在进程表中。

把进程的数据（寄存器中的数据）入栈，放入进程表中的堆栈中。

#### 从进程栈到内核栈

在时钟中断例程中，保护进程调度算法，哪怕只是打印一个字符串，都可能会用到堆栈。为了避免进程的堆栈被破坏，索性将堆栈切换到内核栈，然后再写进程调度算法等。在刚进入内核时，就切换了GDT和堆栈。在这里，才明白为什么进入内核后要切换堆栈。

#### 从内核栈到进程栈

进程调度结束后，在中断处理程序的末尾，要再选择一个进程去执行。方法是：

1. 把sp指向当前进程体的堆栈的最低地址处sp_lowest。
2. 把tss的sp0指向当前进程体的堆栈的最高地址处sp_highest。
3. 从sp_lowest开始，手工依次出栈恢复寄存器的值。
4. 最后使用iretd出栈`eip、cs、eflags、esp、ss`。
   1. `eip、cs、eflags、esp、ss`是`call`入栈的，顺序是``eip、cs、eflags、esp、ss`的倒序。

## hwint00

```assembly
ALIGN	16
hwint00:		; Interrupt routine for irq 0 (the clock).
	sub	esp, 4
	pushad		; `.
	push	ds	;  |
	push	es	;  | 保存原寄存器值
	push	fs	;  |
	push	gs	; /
	mov	dx, ss
	mov	ds, dx
	mov	es, dx

	inc	byte [gs:0]		; 改变屏幕第 0 行, 第 0 列的字符

	mov	al, EOI			; `. reenable
	out	INT_M_CTL, al		; /  master 8259

	inc	dword [k_reenter]
	cmp	dword [k_reenter], 0
	jne	.1	; 重入时跳到.1，通常情况下顺序执行
	;jne	.re_enter
	
	mov	esp, StackTop		; 切到内核栈

	push	.restart_v2
	jmp	.2
.1:					; 中断重入
	push	.restart_reenter_v2
.2:					; 没有中断重入
	sti
	
	push	0					
	call	clock_handler
	add	esp, 4
	
	cli

	ret	; 重入时跳到.restart_reenter_v2，通常情况下到.restart_v2

.restart_v2:
	mov	esp, [p_proc_ready]	; 离开内核栈
	lldt	[esp + P_LDT_SEL]
	lea	eax, [esp + P_STACKTOP]
	mov	dword [tss + TSS3_S_SP0], eax

.restart_reenter_v2:			; 如果(k_reenter != 0)，会跳转到这里
;.re_enter:	; 如果(k_reenter != 0)，会跳转到这里
	dec	dword [k_reenter]
	pop	gs	; `.
	pop	fs	;  |
	pop	es	;  | 恢复原寄存器值
	pop	ds	;  |
	popad		; /
	add	esp, 4

	iretd
```

```assembly
cmp	dword [k_reenter], 0
	jne	.1	; 重入时跳到.1，通常情况下顺序执行
	;jne	.re_enter
	
	mov	esp, StackTop		; 切到内核栈

	push	.restart_v2
	jmp	.2
.1:					; 中断重入
	push	.restart_reenter_v2
.2:					; 没有中断重入
	sti
	
	push	0					
	call	clock_handler
	add	esp, 4
	
	cli

	ret	; 重入时跳到.restart_reenter_v2，通常情况下到.restart_v2
```

理解这段的执行流程，颇为费时。

```assembly
mov	esp, StackTop		; 切到内核栈

push	.restart_v2
```

把`restart_v2`压入内核栈。

