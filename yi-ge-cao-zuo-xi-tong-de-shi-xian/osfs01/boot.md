# boot

```assembly
	; 编译成机器码之后，第一条指令的内存地址是07c00h。h是16进制标志，也可以写成0x7c00。
	org	07c00h			; where the code will be running
	; cs存储代码段，ax是通用寄存器，把代码段地址放到ax中。
	mov	ax, cs
	; ds是数据段。
	mov	ds, ax
	; es是啥？
	mov	es, ax
	;上面三句，把ds、es都赋值为cs，即代码段、数据段、es（？）都指向同一个内存地址。为啥要指向同一个地址？我不明白。
	; 调用函数DispStr
	call	DispStr			; let's display a string
	; $ 是当前位置，跳转到当前位置，即死循环
	jmp	$			; and loop forever
DispStr:
	mov	ax, BootMessage
	mov	bp, ax			; ES:BP = string address
	; CX = 串长度
	mov	cx, 16			; CX = string length
	; AH = 13，显示字符串
	; 作用是什么？
	mov	ax, 01301h		; AH = 13,  AL = 01h
	mov	bx, 000ch		; RED/BLACK
	; DH， DL = 起始行列
	mov	dl, 0
	; int 10h是
	int	10h
	ret
BootMessage:		db	"Hello, OS world!"
; $ 是当前位置，$$ 是这个段的起始位置
times 	510-($-$$)	db	0	; fill zeros to make it exactly 512 bytes
; dw 是一个字，即两个字节。0xaa55是魔数，小端法。
dw 	0xaa55				; boot record signature
```



一般把数据段的段地址放在 DS 寄存器中，当然，如果你硬要觉得 DS 不顺眼，那你可以换个 ES 也是一样的，至于 ES（Extra Segment） 段寄存器的话，自然，是一个附加段寄存器，如果再说得过分点，就当它是个扩展吧，当你发现，你几个段寄存器不够用的时候，你可以考虑使用  ES 段寄存器。

bp为基址寄存器，一般在函数中用来保存进入函数时的sp的栈顶基址。每次子函数调用时，系统在开始时都会保存这个两个指针并在函数结束时恢复sp和bp的值。

AX、BX、CX、DX是CPU内部的通用寄存器中的数据寄存器助记符。






