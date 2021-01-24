# lib.inc

## 代码

```assembly
;; lib.inc

;; 显示 AL 中的数字
DispAL:
	push	ecx
	push	edx
	push	edi

	mov	edi, [dwDispPos]

	mov	ah, 0Fh			; 0000b: 黑底    1111b: 白字
	mov	dl, al
	; 4被显示为04，A被显示为0A
	; al的高4位
	shr	al, 4
	; 8位被等分成2份
	mov	ecx, 2
.begin:
	; 保留al的低4位
	and	al, 01111b
	cmp	al, 9
	; al 大于 9，跳转到 .1
	ja	.1
	add	al, '0'
	jmp	.2
.1:
	; 大于9的，显示为十六进制
	sub	al, 0Ah
	add	al, 'A'
.2:
	mov	[gs:edi], ax
	; 每个字符用2个字节显示，分别是属性和值
	add	edi, 2
	
	; 原始al的低四位
	mov	al, dl
	loop	.begin
	;add	edi, 2

	mov	[dwDispPos], edi

	pop	edi
	pop	edx
	pop	ecx

	ret
;; DispAL 结束


;; 显示一个整型数
DispInt:
	; 分别显示整数的第31~24，编号从0开始
	mov	eax, [esp + 4]
	shr	eax, 24
	call	DispAL
	
	mov	eax, [esp + 4]
	;整数的第31~16，前面的第31~24如何处理？在DispAL中处理，它只处理8位
	shr	eax, 16
	call	DispAL

	mov	eax, [esp + 4]
	shr	eax, 8
	call	DispAL

	mov	eax, [esp + 4]
	call	DispAL

	mov	ah, 07h			; 0000b: 黑底    0111b: 灰字
	mov	al, 'h'
	push	edi
	mov	edi, [dwDispPos]
	mov	[gs:edi], ax
	add	edi, 4
	mov	[dwDispPos], edi
	pop	edi

	ret
;; DispInt 结束

;; 显示一个字符串
DispStr:
	push	ebp
	mov	ebp, esp
	push	ebx
	push	esi
	push	edi

	mov	esi, [ebp + 8]	; pszInfo
	mov	edi, [dwDispPos]
	mov	ah, 0Fh
.1:
	; 把 es:esi中的数据复制到al中
	lodsb
	test	al, al
	jz	.2
	cmp	al, 0Ah	; 是换行键吗?
	jnz	.3
	; 包含除法指令 div ，不理解start
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
	; 包含除法指令 div ，不理解end
	jmp	.1
.3:
	mov	[gs:edi], ax
	add	edi, 2
	jmp	.1

.2:
	mov	[dwDispPos], edi

	pop	edi
	pop	esi
	pop	ebx
	pop	ebp
	ret
;; DispStr 结束

;; 换行
DispReturn:
	push	szReturn
	call	DispStr			;printf("\n");
	add	esp, 4

	ret
;; DispReturn 结束
```

