# undefined

## 代码

### Makefile

```makefile
##################################################
# Makefile
##################################################

BOOT:=boot.asm
LDR:=loader.asm
KERNEL:=kernel.asm
BOOT_BIN:=$(subst .asm,.bin,$(BOOT))
LDR_BIN:=$(subst .asm,.bin,$(LDR))
KERNEL_BIN:=$(subst .asm,.bin,$(KERNEL))

IMG:=a.img
FLOPPY:=/mnt/floppy/

.PHONY : everything

everything : $(BOOT_BIN) $(LDR_BIN) $(KERNEL_BIN)
	dd if=$(BOOT_BIN) of=$(IMG) bs=512 count=1 conv=notrunc
	sudo mount -o loop $(IMG) $(FLOPPY)
	sudo cp $(LDR_BIN) $(FLOPPY) -v
	sudo cp $(KERNEL_BIN) $(FLOPPY) -v
	sudo umount $(FLOPPY)

clean :
	rm -f $(BOOT_BIN) $(LDR_BIN) $(KERNEL_BIN) *.o

$(BOOT_BIN) : $(BOOT)
	nasm $< -o $@

$(LDR_BIN) : $(LDR)
	nasm $< -o $@

$(KERNEL_BIN) : $(KERNEL)
	nasm -f elf -o $(subst .asm,.o,$(KERNEL)) $<
	ld -s -o $@ $(subst .asm,.o,$(KERNEL))
```

### loader.asm

```assembly
org  0100h

BaseOfStack		equ	0100h

BaseOfKernelFile	equ	 08000h	; KERNEL.BIN 被加载到的位置 ----  段地址
OffsetOfKernelFile	equ	     0h	; KERNEL.BIN 被加载到的位置 ---- 偏移地址


	jmp	LABEL_START		; Start

; 下面是 FAT12 磁盘的头, 之所以包含它是因为下面用到了磁盘的一些信息
%include	"fat12hdr.inc"


LABEL_START:			; <--- 从这里开始 *************
	mov	ax, cs
	mov	ds, ax
	mov	es, ax
	mov	ss, ax
	mov	sp, BaseOfStack

	mov	dh, 0			; "Loading  "
	call	DispStr			; 显示字符串

	; 下面在 A 盘的根目录寻找 KERNEL.BIN
	mov	word [wSectorNo], SectorNoOfRootDirectory	
	xor	ah, ah	; `.
	xor	dl, dl	;  | 软驱复位
	int	13h	; /
LABEL_SEARCH_IN_ROOT_DIR_BEGIN:
	cmp	word [wRootDirSizeForLoop], 0	; `.
	jz	LABEL_NO_KERNELBIN		;  | 判断根目录区是不是已经读完,
	dec	word [wRootDirSizeForLoop]	; /  读完表示没有找到 KERNEL.BIN
	mov	ax, BaseOfKernelFile
	mov	es, ax			; es <- BaseOfKernelFile
	mov	bx, OffsetOfKernelFile	; bx <- OffsetOfKernelFile
	mov	ax, [wSectorNo]		; ax <- Root Directory 中的某 Sector 号
	mov	cl, 1
	call	ReadSector

	mov	si, KernelFileName	; ds:si -> "KERNEL  BIN"
	mov	di, OffsetOfKernelFile
	cld
	mov	dx, 10h
LABEL_SEARCH_FOR_KERNELBIN:
	cmp	dx, 0				  ; `.
	jz	LABEL_GOTO_NEXT_SECTOR_IN_ROOT_DIR;  | 循环次数控制, 如果已经读完
	dec	dx				  ; /  了一个 Sector, 就跳到下一个
	mov	cx, 11
LABEL_CMP_FILENAME:
	cmp	cx, 0			; `.
	jz	LABEL_FILENAME_FOUND	;  | 循环次数控制, 如果比较了 11 个字符都
	dec	cx			; /  相等, 表示找到
	lodsb				; ds:si -> al
	cmp	al, byte [es:di]	; if al == es:di
	jz	LABEL_GO_ON
	jmp	LABEL_DIFFERENT
LABEL_GO_ON:
	inc	di
	jmp	LABEL_CMP_FILENAME	;	继续循环

LABEL_DIFFERENT:
	and	di, 0FFE0h		; else`. 让 di 是 20h 的倍数
	add	di, 20h			;      |
	mov	si, KernelFileName	;      | di += 20h  下一个目录条目
	jmp	LABEL_SEARCH_FOR_KERNELBIN;   /

LABEL_GOTO_NEXT_SECTOR_IN_ROOT_DIR:
	add	word [wSectorNo], 1
	jmp	LABEL_SEARCH_IN_ROOT_DIR_BEGIN

LABEL_NO_KERNELBIN:
	mov	dh, 2			; "No KERNEL."
	call	DispStr			; 显示字符串
%ifdef	_LOADER_DEBUG_
	mov	ax, 4c00h		; `.
	int	21h			; / 没有找到 KERNEL.BIN, 回到 DOS
%else
	jmp	$			; 没有找到 KERNEL.BIN, 死循环在这里
%endif

LABEL_FILENAME_FOUND:			; 找到 KERNEL.BIN 后便来到这里继续
	mov	ax, RootDirSectors
	and	di, 0FFF0h		; di -> 当前条目的开始

	push	eax
	mov	eax, [es : di + 01Ch]		; `.
	mov	dword [dwKernelSize], eax	; / 保存 KERNEL.BIN 文件大小
	pop	eax

	add	di, 01Ah		; di -> 首 Sector
	mov	cx, word [es:di]
	push	cx			; 保存此 Sector 在 FAT 中的序号
	add	cx, ax
	add	cx, DeltaSectorNo	; cl <- KERNEL.BIN 的起始扇区号(0-based)
	mov	ax, BaseOfKernelFile
	mov	es, ax			; es <- BaseOfKernelFile
	mov	bx, OffsetOfKernelFile	; bx <- OffsetOfKernelFile
	mov	ax, cx			; ax <- Sector 号

LABEL_GOON_LOADING_FILE:
	push	ax			; `.
	push	bx			;  |
	mov	ah, 0Eh			;  | 每读一个扇区就在 "Loading  " 后面
	mov	al, '.'			;  | 打一个点, 形成这样的效果:
	mov	bl, 0Fh			;  | Loading ......
	int	10h			;  |
	pop	bx			;  |
	pop	ax			; /

	mov	cl, 1
	call	ReadSector
	pop	ax			; 取出此 Sector 在 FAT 中的序号
	call	GetFATEntry
	cmp	ax, 0FFFh
	jz	LABEL_FILE_LOADED
	push	ax			; 保存 Sector 在 FAT 中的序号
	mov	dx, RootDirSectors
	add	ax, dx
	add	ax, DeltaSectorNo
	add	bx, [BPB_BytsPerSec]
	jmp	LABEL_GOON_LOADING_FILE
LABEL_FILE_LOADED:

	call	KillMotor		; 关闭软驱马达

	mov	dh, 1			; "Ready."
	call	DispStr			; 显示字符串

	jmp	$


;============================================================================
;变量
;----------------------------------------------------------------------------
wRootDirSizeForLoop	dw	RootDirSectors	; Root Directory 占用的扇区数
wSectorNo		dw	0		; 要读取的扇区号
bOdd			db	0		; 奇数还是偶数
dwKernelSize		dd	0		; KERNEL.BIN 文件大小

;============================================================================
;字符串
;----------------------------------------------------------------------------
KernelFileName		db	"KERNEL  BIN", 0	; KERNEL.BIN 之文件名
; 为简化代码, 下面每个字符串的长度均为 MessageLength
MessageLength		equ	9
LoadMessage:		db	"Loading  "
Message1		db	"Ready.   "
Message2		db	"No KERNEL"
;============================================================================

;----------------------------------------------------------------------------
; 函数名: DispStr
;----------------------------------------------------------------------------
; 作用:
;	显示一个字符串, 函数开始时 dh 中应该是字符串序号(0-based)
DispStr:
	mov	ax, MessageLength
	mul	dh
	add	ax, LoadMessage
	mov	bp, ax			; ┓
	mov	ax, ds			; ┣ ES:BP = 串地址
	mov	es, ax			; ┛
	mov	cx, MessageLength	; CX = 串长度
	mov	ax, 01301h		; AH = 13,  AL = 01h
	mov	bx, 0007h		; 页号为0(BH = 0) 黑底白字(BL = 07h)
	mov	dl, 0
	add	dh, 3			; 从第 3 行往下显示
	int	10h			; int 10h
	ret
;----------------------------------------------------------------------------
; 函数名: ReadSector
;----------------------------------------------------------------------------
; 作用:
;	从序号(Directory Entry 中的 Sector 号)为 ax 的的 Sector 开始, 将 cl 个 Sector 读入 es:bx 中
ReadSector:
	; -----------------------------------------------------------------------
	; 怎样由扇区号求扇区在磁盘中的位置 (扇区号 -> 柱面号, 起始扇区, 磁头号)
	; -----------------------------------------------------------------------
	; 设扇区号为 x
	;                           ┌ 柱面号 = y >> 1
	;       x           ┌ 商 y ┤
	; -------------- => ┤      └ 磁头号 = y & 1
	;  每磁道扇区数     │
	;                   └ 余 z => 起始扇区号 = z + 1
	push	bp
	mov	bp, sp
	sub	esp, 2			; 辟出两个字节的堆栈区域保存要读的扇区数: byte [bp-2]

	mov	byte [bp-2], cl
	push	bx			; 保存 bx
	mov	bl, [BPB_SecPerTrk]	; bl: 除数
	div	bl			; y 在 al 中, z 在 ah 中
	inc	ah			; z ++
	mov	cl, ah			; cl <- 起始扇区号
	mov	dh, al			; dh <- y
	shr	al, 1			; y >> 1 (其实是 y/BPB_NumHeads, 这里BPB_NumHeads=2)
	mov	ch, al			; ch <- 柱面号
	and	dh, 1			; dh & 1 = 磁头号
	pop	bx			; 恢复 bx
	; 至此, "柱面号, 起始扇区, 磁头号" 全部得到 ^^^^^^^^^^^^^^^^^^^^^^^^
	mov	dl, [BS_DrvNum]		; 驱动器号 (0 表示 A 盘)
.GoOnReading:
	mov	ah, 2			; 读
	mov	al, byte [bp-2]		; 读 al 个扇区
	int	13h
	jc	.GoOnReading		; 如果读取错误 CF 会被置为 1, 这时就不停地读, 直到正确为止

	add	esp, 2
	pop	bp

	ret

;----------------------------------------------------------------------------
; 函数名: GetFATEntry
;----------------------------------------------------------------------------
; 作用:
;	找到序号为 ax 的 Sector 在 FAT 中的条目, 结果放在 ax 中
;	需要注意的是, 中间需要读 FAT 的扇区到 es:bx 处, 所以函数一开始保存了 es 和 bx
GetFATEntry:
	push	es
	push	bx
	push	ax
	mov	ax, BaseOfKernelFile	; ┓
	sub	ax, 0100h		; ┣ 在 BaseOfKernelFile 后面留出 4K 空间用于存放 FAT
	mov	es, ax			; ┛
	pop	ax
	mov	byte [bOdd], 0
	mov	bx, 3
	mul	bx			; dx:ax = ax * 3
	mov	bx, 2
	div	bx			; dx:ax / 2  ==>  ax <- 商, dx <- 余数
	cmp	dx, 0
	jz	LABEL_EVEN
	mov	byte [bOdd], 1
LABEL_EVEN:;偶数
	xor	dx, dx			; 现在 ax 中是 FATEntry 在 FAT 中的偏移量. 下面来计算 FATEntry 在哪个扇区中(FAT占用不止一个扇区)
	mov	bx, [BPB_BytsPerSec]
	div	bx			; dx:ax / BPB_BytsPerSec  ==>	ax <- 商   (FATEntry 所在的扇区相对于 FAT 来说的扇区号)
					;				dx <- 余数 (FATEntry 在扇区内的偏移)。
	push	dx
	mov	bx, 0			; bx <- 0	于是, es:bx = (BaseOfKernelFile - 100):00 = (BaseOfKernelFile - 100) * 10h
	add	ax, SectorNoOfFAT1	; 此句执行之后的 ax 就是 FATEntry 所在的扇区号
	mov	cl, 2
	call	ReadSector		; 读取 FATEntry 所在的扇区, 一次读两个, 避免在边界发生错误, 因为一个 FATEntry 可能跨越两个扇区
	pop	dx
	add	bx, dx
	mov	ax, [es:bx]
	cmp	byte [bOdd], 1
	jnz	LABEL_EVEN_2
	shr	ax, 4
LABEL_EVEN_2:
	and	ax, 0FFFh

LABEL_GET_FAT_ENRY_OK:

	pop	bx
	pop	es
	ret
;----------------------------------------------------------------------------


;----------------------------------------------------------------------------
; 函数名: KillMotor
;----------------------------------------------------------------------------
; 作用:
;	关闭软驱马达
KillMotor:
	push	dx
	mov	dx, 03F2h
	mov	al, 0
	out	dx, al
	pop	dx
	ret
;----------------------------------------------------------------------------
```



### kernel.asm

```assembly
; 编译链接方法
; $ nasm -f elf kernel.asm -o kernel.o
; $ ld -s kernel.o -o kernel.bin    #‘-s’选项意为“strip all”

[section .text]	; 代码在此

global _start	; 导出 _start

_start:	; 跳到这里来的时候，我们假设 gs 指向显存
	mov	ah, 0Fh				; 0000: 黑底    1111: 白字
	mov	al, 'K'
	mov	[gs:((80 * 1 + 39) * 2)], ax	; 屏幕第 1 行, 第 39 列
	jmp	$
```



### boot.asm

```assembly

;%define	_BOOT_DEBUG_	; 做 Boot Sector 时一定将此行注释掉!将此行打开后用 nasm Boot.asm -o Boot.com 做成一个.COM文件易于调试

%ifdef	_BOOT_DEBUG_
	org  0100h			; 调试状态, 做成 .COM 文件, 可调试
%else
	org  07c00h			; Boot 状态, Bios 将把 Boot Sector 加载到 0:7C00 处并开始执行
%endif

;================================================================================================
%ifdef	_BOOT_DEBUG_
BaseOfStack		equ	0100h	; 调试状态下堆栈基地址(栈底, 从这个位置向低地址生长)
%else
BaseOfStack		equ	07c00h	; Boot状态下堆栈基地址(栈底, 从这个位置向低地址生长)
%endif
; 这个值根据什么标准设置的？
BaseOfLoader		equ	09000h	; LOADER.BIN 被加载到的位置 ----  段地址
OffsetOfLoader		equ	0100h	; LOADER.BIN 被加载到的位置 ---- 偏移地址
;================================================================================================

	jmp short LABEL_START		; Start to boot.
	nop				; 这个 nop 不可少

; 下面是 FAT12 磁盘的头, 之所以包含它是因为下面用到了磁盘的一些信息
; 注意，引入使用 %include
%include	"fat12hdr.inc"

LABEL_START:
	mov	ax, cs
	mov	ds, ax
	mov	es, ax
	mov	ss, ax
	; 上面的语句，将 ds、es、ss 等都设置为cs，即代码段。
	mov	sp, BaseOfStack

	; 清屏。很多不理解。
	mov	ax, 0600h		; AH = 6,  AL = 0h
	mov	bx, 0700h		; 黑底白字(BL = 07h)。理解。
	mov	cx, 0			; 左上角: (0, 0)
	mov	dx, 0184fh		; 右下角: (80, 50)
	int	10h			; int 10h。理解。

	mov	dh, 0			; "Booting  "
	call	DispStr			; 显示字符串
	
	xor	ah, ah	; ┓
	xor	dl, dl	; ┣ 软驱复位。模板。
	int	13h	; ┛
	
; 下面在 A 盘的根目录寻找 LOADER.BIN
	mov	word [wSectorNo], SectorNoOfRootDirectory
LABEL_SEARCH_IN_ROOT_DIR_BEGIN:
	cmp	word [wRootDirSizeForLoop], 0	; ┓
	jz	LABEL_NO_LOADERBIN		; ┣ 判断根目录区是不是已经读完
	dec	word [wRootDirSizeForLoop]	; ┛ 如果读完表示没有找到 LOADER.BIN
	mov	ax, BaseOfLoader
	mov	es, ax			; es <- BaseOfLoader
	mov	bx, OffsetOfLoader	; bx <- OffsetOfLoader	于是, es:bx = BaseOfLoader:OffsetOfLoader
	mov	ax, [wSectorNo]	; ax <- Root Directory 中的某 Sector 号
	mov	cl, 1
	call	ReadSector

	mov	si, LoaderFileName	; ds:si -> "LOADER  BIN"
	mov	di, OffsetOfLoader	; es:di -> BaseOfLoader:0100 = BaseOfLoader*10h+100
	cld
	mov	dx, 10h
LABEL_SEARCH_FOR_LOADERBIN:
	cmp	dx, 0										; ┓循环次数控制,
	jz	LABEL_GOTO_NEXT_SECTOR_IN_ROOT_DIR	; ┣如果已经读完了一个 Sector,
	dec	dx											; ┛就跳到下一个 Sector
	mov	cx, 11
LABEL_CMP_FILENAME:
	cmp	cx, 0
	jz	LABEL_FILENAME_FOUND	; 如果比较了 11 个字符都相等, 表示找到
dec	cx
	lodsb				; ds:si -> al
	cmp	al, byte [es:di]
	jz	LABEL_GO_ON
	jmp	LABEL_DIFFERENT		; 只要发现不一样的字符就表明本 DirectoryEntry 不是
; 我们要找的 LOADER.BIN
LABEL_GO_ON:
	inc	di
	jmp	LABEL_CMP_FILENAME	;	继续循环

LABEL_DIFFERENT:
	and	di, 0FFE0h						; else ┓	di &= E0 为了让它指向本条目开头
	add	di, 20h							;     ┃
	mov	si, LoaderFileName					;     ┣ di += 20h  下一个目录条目
	jmp	LABEL_SEARCH_FOR_LOADERBIN;    ┛

LABEL_GOTO_NEXT_SECTOR_IN_ROOT_DIR:
	add	word [wSectorNo], 1
	jmp	LABEL_SEARCH_IN_ROOT_DIR_BEGIN

LABEL_NO_LOADERBIN:
	mov	dh, 2			; "No LOADER."
	call	DispStr			; 显示字符串
%ifdef	_BOOT_DEBUG_
	mov	ax, 4c00h		; ┓
	int	21h			; ┛没有找到 LOADER.BIN, 回到 DOS
%else
	jmp	$			; 没有找到 LOADER.BIN, 死循环在这里
%endif

LABEL_FILENAME_FOUND:			; 找到 LOADER.BIN 后便来到这里继续
	mov	ax, RootDirSectors
	and	di, 0FFE0h		; di -> 当前条目的开始
	add	di, 01Ah		; di -> 首 Sector
	mov	cx, word [es:di]
	push	cx			; 保存此 Sector 在 FAT 中的序号
	add	cx, ax
	add	cx, DeltaSectorNo	; 这句完成时 cl 里面变成 LOADER.BIN 的起始扇区号 (从 0 开始数的序号)
	mov	ax, BaseOfLoader
	mov	es, ax			; es <- BaseOfLoader
	mov	bx, OffsetOfLoader	; bx <- OffsetOfLoader	于是, es:bx = BaseOfLoader:OffsetOfLoader = BaseOfLoader * 10h + OffsetOfLoader
	mov	ax, cx			; ax <- Sector 号

LABEL_GOON_LOADING_FILE:
	push	ax			; ┓
	push	bx			; ┃
	mov	ah, 0Eh			; ┃ 每读一个扇区就在 "Booting  " 后面打一个点, 形成这样的效果:
	mov	al, '.'			; ┃
	mov	bl, 0Fh			; ┃ Booting ......
	int	10h			; ┃
	pop	bx			; ┃
	pop	ax			; ┛

	mov	cl, 1
	call	ReadSector
	pop	ax			; 取出此 Sector 在 FAT 中的序号
	call	GetFATEntry
	cmp	ax, 0FFFh
	jz	LABEL_FILE_LOADED
	push	ax			; 保存 Sector 在 FAT 中的序号
	mov	dx, RootDirSectors
	add	ax, dx
	add	ax, DeltaSectorNo
	add	bx, [BPB_BytsPerSec]
	jmp	LABEL_GOON_LOADING_FILE
LABEL_FILE_LOADED:

	mov	dh, 1			; "Ready."
	call	DispStr			; 显示字符串

; *****************************************************************************************************
	jmp	BaseOfLoader:OffsetOfLoader	; 这一句正式跳转到已加载到内存中的 LOADER.BIN 的开始处
						; 开始执行 LOADER.BIN 的代码
						; Boot Sector 的使命到此结束
; *****************************************************************************************************



;============================================================================
;变量
;----------------------------------------------------------------------------
wRootDirSizeForLoop	dw	RootDirSectors	; Root Directory 占用的扇区数, 在循环中会递减至零.
wSectorNo		dw	0		; 要读取的扇区号
bOdd			db	0		; 奇数还是偶数

;============================================================================
;字符串
;----------------------------------------------------------------------------
LoaderFileName		db	"LOADER  BIN", 0	; LOADER.BIN 之文件名
; 为简化代码, 下面每个字符串的长度均为 MessageLength
MessageLength		equ	9
BootMessage:		db	"Booting  "; 9字节, 不够则用空格补齐. 序号 0
Message1		db	"Ready.   "; 9字节, 不够则用空格补齐. 序号 1
Message2		db	"No LOADER"; 9字节, 不够则用空格补齐. 序号 2
;============================================================================


;----------------------------------------------------------------------------
; 函数名: DispStr
;----------------------------------------------------------------------------
; 作用:
;	显示一个字符串, 函数开始时 dh 中应该是字符串序号(0-based)
DispStr:
	mov	ax, MessageLength
	mul	dh
	add	ax, BootMessage
	mov	bp, ax			; ┓
	mov	ax, ds			; ┣ ES:BP = 串地址
	mov	es, ax			; ┛
	mov	cx, MessageLength	; CX = 串长度
	mov	ax, 01301h		; AH = 13,  AL = 01h
	mov	bx, 0007h		; 页号为0(BH = 0) 黑底白字(BL = 07h)
	mov	dl, 0
	int	10h			; int 10h
	ret


;----------------------------------------------------------------------------
; 函数名: ReadSector
;----------------------------------------------------------------------------
; 作用:
;	从第 ax 个 Sector 开始, 将 cl 个 Sector 读入 es:bx 中
ReadSector:
	; -----------------------------------------------------------------------
	; 怎样由扇区号求扇区在磁盘中的位置 (扇区号 -> 柱面号, 起始扇区, 磁头号)
	; -----------------------------------------------------------------------
	; 设扇区号为 x
	;                           ┌ 柱面号 = y >> 1
	;       x           ┌ 商 y ┤
	; -------------- => ┤      └ 磁头号 = y & 1
	;  每磁道扇区数     │
	;                   └ 余 z => 起始扇区号 = z + 1
	push	bp
	mov	bp, sp
	sub	esp, 2			; 辟出两个字节的堆栈区域保存要读的扇区数: byte [bp-2]

	mov	byte [bp-2], cl
	push	bx			; 保存 bx
	mov	bl, [BPB_SecPerTrk]	; bl: 除数
	div	bl			; y 在 al 中, z 在 ah 中
	inc	ah			; z ++
	mov	cl, ah			; cl <- 起始扇区号
	mov	dh, al			; dh <- y
	shr	al, 1			; y >> 1 (其实是 y/BPB_NumHeads, 这里BPB_NumHeads=2)
	mov	ch, al			; ch <- 柱面号
	and	dh, 1			; dh & 1 = 磁头号
	pop	bx			; 恢复 bx
	; 至此, "柱面号, 起始扇区, 磁头号" 全部得到 ^^^^^^^^^^^^^^^^^^^^^^^^
	mov	dl, [BS_DrvNum]		; 驱动器号 (0 表示 A 盘)
.GoOnReading:
	mov	ah, 2			; 读
	mov	al, byte [bp-2]		; 读 al 个扇区
	int	13h
	jc	.GoOnReading		; 如果读取错误 CF 会被置为 1, 这时就不停地读, 直到正确为止

	add	esp, 2
	pop	bp

	ret

;----------------------------------------------------------------------------
; 函数名: GetFATEntry
;----------------------------------------------------------------------------
; 理解不了。
; 作用:
;	找到序号为 ax 的 Sector 在 FAT 中的条目, 结果放在 ax 中
;	需要注意的是, 中间需要读 FAT 的扇区到 es:bx 处, 所以函数一开始保存了 es 和 bx
GetFATEntry:
	push	es
	push	bx
	push	ax
	mov	ax, BaseOfLoader	; ┓
	sub	ax, 0100h		; ┣ 在 BaseOfLoader 后面留出 4K 空间用于存放 FAT
	mov	es, ax			; ┛
	pop	ax
	mov	byte [bOdd], 0
	mov	bx, 3
	mul	bx			; dx:ax = ax * 3
	mov	bx, 2
	div	bx			; dx:ax / 2  ==>  ax <- 商, dx <- 余数
	cmp	dx, 0
	jz	LABEL_EVEN
	mov	byte [bOdd], 1
LABEL_EVEN:;偶数
	xor	dx, dx			; 现在 ax 中是 FATEntry 在 FAT 中的偏移量. 下面来计算 FATEntry 在哪个扇区中(FAT占用不止一个扇区)
	mov	bx, [BPB_BytsPerSec]
	div	bx			; dx:ax / BPB_BytsPerSec  ==>	ax <- 商   (FATEntry 所在的扇区相对于 FAT 来说的扇区号)
					;				dx <- 余数 (FATEntry 在扇区内的偏移)。
	push	dx
	mov	bx, 0			; bx <- 0	于是, es:bx = (BaseOfLoader - 100):00 = (BaseOfLoader - 100) * 10h
	add	ax, SectorNoOfFAT1	; 此句执行之后的 ax 就是 FATEntry 所在的扇区号
	mov	cl, 2
	call	ReadSector		; 读取 FATEntry 所在的扇区, 一次读两个, 避免在边界发生错误, 因为一个 FATEntry 可能跨越两个扇区
	pop	dx
	add	bx, dx
	mov	ax, [es:bx]
	cmp	byte [bOdd], 1
	jnz	LABEL_EVEN_2
	shr	ax, 4
LABEL_EVEN_2:
	and	ax, 0FFFh

LABEL_GET_FAT_ENRY_OK:

	pop	bx
	pop	es
	ret
;----------------------------------------------------------------------------

times 	510-($-$$)	db	0	; 填充剩下的空间，使生成的二进制代码恰好为512字节
dw 	0xaa55				; 结束标志
```



### fat12hdr.inc

```assembly
; FAT12 磁盘的头
; 在软盘上按FAT12格式存储数据。前面的标签是固定的，后面的值又要求吗？还是可以自己设置？
; ----------------------------------------------------------------------
BS_OEMName	DB 'ForrestY'	; OEM String, 必须 8 个字节

BPB_BytsPerSec	DW 512		; 每扇区字节数
BPB_SecPerClus	DB 1		; 每簇多少扇区
BPB_RsvdSecCnt	DW 1		; Boot 记录占用多少扇区
BPB_NumFATs	DB 2		; 共有多少 FAT 表
BPB_RootEntCnt	DW 224		; 根目录文件数最大值
BPB_TotSec16	DW 2880		; 逻辑扇区总数
BPB_Media	DB 0xF0		; 媒体描述符
BPB_FATSz16	DW 9		; 每FAT扇区数
BPB_SecPerTrk	DW 18		; 每磁道扇区数
BPB_NumHeads	DW 2		; 磁头数(面数)
BPB_HiddSec	DD 0		; 隐藏扇区数
BPB_TotSec32	DD 0		; 如果 wTotalSectorCount 是 0 由这个值记录扇区数

BS_DrvNum	DB 0		; 中断 13 的驱动器号
BS_Reserved1	DB 0		; 未使用
BS_BootSig	DB 29h		; 扩展引导标记 (29h)
BS_VolID	DD 0		; 卷序列号
BS_VolLab	DB 'OrangeS0.02'; 卷标, 必须 11 个字节
BS_FileSysType	DB 'FAT12   '	; 文件系统类型, 必须 8个字节  
;------------------------------------------------------------------------


; -------------------------------------------------------------------------
; 基于 FAT12 头的一些常量定义，如果头信息改变，下面的常量可能也要做相应改变
; -------------------------------------------------------------------------
; BPB_FATSz16
FATSz			equ	9

; 根目录占用空间:
; RootDirSectors = ((BPB_RootEntCnt*32)+(BPB_BytsPerSec–1))/BPB_BytsPerSec
; 但如果按照此公式代码过长，故定义此宏
RootDirSectors		equ	14

; Root Directory 的第一个扇区号	= BPB_RsvdSecCnt + (BPB_NumFATs * FATSz)
SectorNoOfRootDirectory	equ	19

; FAT1 的第一个扇区号	= BPB_RsvdSecCnt
SectorNoOfFAT1		equ	1

; DeltaSectorNo = BPB_RsvdSecCnt + (BPB_NumFATs * FATSz) - 2
; 文件的开始Sector号 = DirEntry中的开始Sector号 + 根目录占用Sector数目
;                      + DeltaSectorNo
; 本该是19，可是第0个、第1个扇区不使用。特指数据区的扇区。
DeltaSectorNo		equ	17
```



### bochsrc

```shell
###############################################################
# Configuration file for Bochs
###############################################################

# how much memory the emulated machine will have
# 该虚拟计算机的内存是32M。
megs: 32

# filename of ROM images
# 不知道具体作用是啥。
romimage: file=/usr/local/share/bochs/BIOS-bochs-latest
vgaromimage: file=/usr/local/share/VGABIOS-lgpl-latest

# what disk images will be used
# 使用软盘a。使用bximage创建软盘时会提示要求将这句写入bochsrc文件。
floppya: 1_44=a.img, status=inserted

# choose the boot disk.
# 设置启动的设备
boot: a

# where do we send log messages?
# log: bochsout.txt

# disable the mouse
# 能使用鼠标？
mouse: enabled=0

# enable key mapping, using US layout as default.
# 作用不明。
keyboard: keymap=/usr/local/share/bochs/keymaps/x11-pc-us.map
```

## 运行

在实体机上使用root用户运行 `make`，然后再虚拟机上运行 `bochs -f bochsrc'。当bochs卡住时，输入 `c	。

## 工具

### xxd

```shell
[root@localhost b]# xxd -u -a -g 1 -c 16 -l 80 foobar
00000000: 7F 45 4C 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
00000010: 02 00 03 00 01 00 00 00 A0 80 04 08 34 00 00 00  ............4...
00000020: 68 10 00 00 00 00 00 00 34 00 20 00 03 00 28 00  h.......4. ...(.
00000030: 07 00 06 00 01 00 00 00 00 00 00 00 00 80 04 08  ................
00000040: 00 80 04 08 64 01 00 00 64 01 00 00 05 00 00 00  ....d...d.......
[root@localhost b]#
```



### readelf

```shell
[root@localhost e]# readelf -h kernel.bin
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Intel 80386
  Version:                           0x1
  Entry point address:               0x30400
  Start of program headers:          52 (bytes into file)
  Start of section headers:          1056 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         1
  Size of section headers:           40 (bytes)
  Number of section headers:         3
  Section header string table index: 2
```

 2's complement，整数二进制补码。

```shell
[root@localhost e]# readelf -l kernel.bin

Elf file type is EXEC (Executable file)
Entry point 0x30400
There is 1 program header, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000000 0x00030000 0x00030000 0x0040d 0x0040d R E 0x1000

 Section to Segment mapping:
  Segment Sections...
   00     .text
```



```shell
[root@localhost e]# readelf -S kernel.bin
There are 3 section headers, starting at offset 0x420:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        00030400 000400 00000d 00  AX  0   0 16
  [ 2] .shstrtab         STRTAB          00000000 00040d 000011 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  p (processor specific)
```

Kernerl.bin 是由下面的代码编译而成的。

```assembly

; 编译链接方法
; [root@XXX XXX]# nasm -f elf kernel.asm -o kernel.o
; [root@XXX XXX]# ld -s -Ttext 0x30400 -o kernel.bin kernel.o
; [root@XXX XXX]# 

[section .text]	; 代码在此

global _start	; 导出 _start

_start:	; 跳到这里来的时候，我们假设 gs 指向显存
	mov	ah, 0Fh				; 0000: 黑底    1111: 白字
	mov	al, 'K'
	mov	[gs:((80 * 1 + 39) * 2)], ax	; 屏幕第 1 行, 第 39 列。
	jmp	$
```

不理解为何用readelf查看有3个section headers。

### nm

```shell
[root@localhost e]# nm kernel.o
00000000 T _start
```



## 临时笔记

啦。就拿 Linux 来说，它的系统调用是先往 eax 寄存器中写入系统调用号，然后通过 Ox80 中断来实现的。

重定位指的是文件里面所用的符号还没有安排地址，这些符号的地址需要将来与其他目标文件“组成”一个可执行文件时再重新定位（编排地址〉，这里的符号就是指该目标文件中所调用的函数或使用的变量，而这里的“组成”就是指链接。这些符号 般是位于其他文件中，所以在编译时不能确定其地址，需要在所有目标文件都到齐了，将它们链接到 起时再重新定位（编排地址）。由于不知道可执行文件由几个目标文件组成，所以 律在链接阶段对符号重新定位（编排地址）。哪怕是可执行文件只是由一个文件组成的，其目标文件中的符号也是未编址的，编址工作，即重定位，一律统一在链接阶段完成。

macbook系统与centos 8系统的可执行文件格式不一样。同样的go代码，在这两个系统编译出的可执行文件，查看格式如下：

```shell
[root@localhost e]# file hello
hello: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, not stripped
```

```shell
chugangdeMacBook-Pro:array cg$ file arr1
arr1: Mach-O 64-bit executable x86_64
```



```shell
ld kernel/main.o -Ttext Oxe0001500 -e main -o kernel/kernel.bin
```

链接器ld默认把函数_start作为程序的入口地址。汇编中能用数字和字符串表示内存地址，所以ld也能用main作为程序的入口地址。

-Ttext指定程序的虚拟内存地址，-e指定程序的入口地址。

### 理解elf

详情参考《操作系统真相还原》5.3.3、5.3.4。

Loader.bin、mbr指定了本程序的入口地址，调用者根据约定好的入口地址调用它们。若不想事先商量好入口地址，可以采用“程序头+程序体”的方式组织mbr等二进制程序。调用者从程序头中读取程序的入口地址，然后将程序体复制到入口地址处（为啥不是直接跳转到入口地址处执行）。下面是为了帮助理解elf而自定义的一种程序。

```assembly
header:
        program_length  dd      program_end - program_start
        start_addr      dd      program_start

body:
program_start:
        mov     ax,     0x1234
        jmp     $
program_end:
```

编译并运行

```assembly
[root@localhost e]# nasm -o header.bin header.s
[root@localhost e]# chmod +x ./header.bin
[root@localhost e]# file header.bin
header.bin: data
[root@localhost e]# ls
Makefile  bochsrc   boot.asm  fat12hdr.inc  header.bin  kernel.asm  kernel.o  loader.asm  pm.inc
a.img     bochsrc2  boot.bin  header        header.s    kernel.bin  load.inc  loader.bin
[root@localhost e]# ./header.bin
-bash: ./header.bin: cannot execute binary file: Exec format error
[root@localhost e]# xxd -u -a -g 1 -c 16 -l 80 header.bin
00000000: 05 00 00 00 08 00 00 00 B8 34 12 EB FE           .........4...
```

虽然header.bin也是二进制文件，但是它的入口地址不是机器指令，所以，不能直接执行。

为啥不是直接跳转到入口地址处执行？若如此，需要在编译时将程序存放到某个物理地址。编译器并不做这个工作，链接器完成这项工作。好像不是不可以。但是，更顺畅的思路应该是，调用者获得被调用者的入口地址后，将被调用者的所有指令复制到入口地址。

Window 下的可执行文件格式是 PE 如果您想说的是 EXE ，不要搞混了， EXE是扩展名，属于文件名的一部分，只是名字的后缀，它并不是真正的格式）， PE Portable Exeeutable, Linux 下可执行文件格式是 ELF 。

ELF 指的是 Exeeutable and Linkable Format ，可执行链接格式。

## 临时笔记

/usr/include/gnu/stubs.h:7:11: fatal error: gnu/stubs-32.h: No such file or directory

centos64位编译32位代码，出现/usr/include/gnu/stubs.h:7:27: 致命错误：gnu/stubs-32.h：没有那个文件或目录，需要安装32位的glibc库文件。

​      安装32位glibc库文件命令：

​      sudo yum install glibc-devel.i686(安装C库文件)

​      sudo dnf install glibc-devel.i686(fedora命令)

​     安装32位glibc++库文件命令

​     sudo yum install libstdc++-devel.i686

​     sudo dnf install libstdc++-devel.i686（fedora命令）

Ubuntu解决命令：

​      sudo apt-get install g++-multilib

## bochs

```shell
b 0x7c00
print-stack
# 执行下一条指令，并打印下一条要执行的命令
s
# u 命令可以使用两个参数，第一个参数是跟在 / 后面地数字，指定返汇编出多少条指令；第二条参数用于指定一个内存地址，Bochs从这里开始反汇编操作。
u/2 0x7c3e
# 打印通用寄存器的数据，ax、bx、cx、dx
r
# 打印段寄存器的数据，cs、es、ss
sreg
xp /32wx 0x7c00
```

```shell
<bochs:4> info gdt
Global Descriptor Table (base=0x00000000000f9af7, limit=48):
GDT[0x0000]=??? descriptor hi=0x00000000, lo=0x00000000
GDT[0x0008]=??? descriptor hi=0x00000000, lo=0x00000000
GDT[0x0010]=Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
GDT[0x0018]=Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
GDT[0x0020]=Code segment, base=0x000f0000, limit=0x0000ffff, Execute/Read, Non-Conforming, Accessed, 16-bit
GDT[0x0028]=Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
You can list individual entries with 'info gdt [NUM]' or groups with 'info gdt [NUM] [NUM]'
```

汇编代码

```assembly
; GDT ------------------------------------------------------------------------------------------------------------------------------------------------------------
;                                                段基址            段界限     , 属性
LABEL_GDT:			Descriptor             0,                    0, 0						; 空描述符
LABEL_DESC_FLAT_C:		Descriptor             0,              0fffffh, DA_CR  | DA_32 | DA_LIMIT_4K			; 0 ~ 4G
LABEL_DESC_FLAT_RW:		Descriptor             0,              0fffffh, DA_DRW | DA_32 | DA_LIMIT_4K			; 0 ~ 4G
LABEL_DESC_VIDEO:		Descriptor	 0B8000h,               0ffffh, DA_DRW                         | DA_DPL3	; 显存首地址
; GDT ------------------------------------------------------------------------------------------------------------------------------------------------------------

GdtLen		equ	$ - LABEL_GDT
GdtPtr		dw	GdtLen - 1				; 段界限
		dd	BaseOfLoaderPhyAddr + LABEL_GDT		; 基地址

; GDT 选择子 ----------------------------------------------------------------------------------
SelectorFlatC		equ	LABEL_DESC_FLAT_C	- LABEL_GDT
SelectorFlatRW		equ	LABEL_DESC_FLAT_RW	- LABEL_GDT
SelectorVideo		equ	LABEL_DESC_VIDEO	- LABEL_GDT + SA_RPL3
; GDT 选择子 ----------------------------------------------------------------------------------
```

只定义了四个GDT，为啥bochs打印出来六个GDT？

我的推测：

```
LABEL_DESC_FLAT_C:		Descriptor             0,              0fffffh, DA_CR  | DA_32 | DA_LIMIT_4K			; 0 ~ 4G
```

对应GDT

```
GDT[0x0010]=Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
```



```
LABEL_DESC_FLAT_RW:		Descriptor             0,              0fffffh, DA_DRW | DA_32 | DA_LIMIT_4K			; 0 ~ 4G
```



对应GDT

```
GDT[0x0018]=Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
```

上面的GDT，是在执行 lgdt [GdtPtr] 前的内容，所以不是在代码中设置的值。执行完 lgdt [GdtPtr] 后，打印gdt中的数据如下：

```
<bochs:6> info gdt
Global Descriptor Table (base=0x000000000009013d, limit=31):
GDT[0x0000]=??? descriptor hi=0x00000000, lo=0x00000000
GDT[0x0008]=Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, 32-bit
GDT[0x0010]=Data segment, base=0x00000000, limit=0xffffffff, Read/Write
GDT[0x0018]=Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write
You can list individual entries with 'info gdt [NUM]' or groups with 'info gdt [NUM] [NUM]'
```

与汇编中创建gdt的代码完全吻合。

```shell
# 查看栈的内容
<bochs:5> print-stack
Stack address size 2
 | STACK 0x90100 [0x61eb] (<unknown>)
 | STACK 0x90102 [0x6f46] (<unknown>)
 | STACK 0x90104 [0x7272] (<unknown>)
 | STACK 0x90106 [0x7365] (<unknown>)
 | STACK 0x90108 [0x5974] (<unknown>)
 | STACK 0x9010a [0x0200] (<unknown>)
 | STACK 0x9010c [0x0101] (<unknown>)
 | STACK 0x9010e [0x0200] (<unknown>)
 | STACK 0x90110 [0x00e0] (<unknown>)
 | STACK 0x90112 [0x0b40] (<unknown>)
 | STACK 0x90114 [0x09f0] (<unknown>)
 | STACK 0x90116 [0x1200] (<unknown>)
 | STACK 0x90118 [0x0200] (<unknown>)
 | STACK 0x9011a [0x0000] (<unknown>)
 | STACK 0x9011c [0x0000] (<unknown>)
 | STACK 0x9011e [0x0000] (<unknown>)
```



```
; 真正进入保护模式
	jmp	dword SelectorFlatC:(BaseOfLoaderPhyAddr+LABEL_PM_START)
```

执行中实际是下面的语句

```
(0) Magic breakpoint
Next at t=16998847
(0) [0x000000090283] 9000:0000000000000283 (unk. ctxt): jmpf 0x0008:00090380      ; 66ea800309000800
```

SelectorFlatC 的地址变成 0x0008，不理解。

00090380 是 BaseOfLoaderPhyAddr + 380 。LABEL_PM_START 在段内的偏移量是 380 吗？不知道如何人工计算。

```
BaseOfLoader		equ	 09000h	; LOADER.BIN 被加载到的位置 ----  段地址
OffsetOfLoader		equ	  0100h	; LOADER.BIN 被加载到的位置 ---- 偏移地址

BaseOfLoaderPhyAddr	equ	BaseOfLoader * 10h	; LOADER.BIN 被加载到的位置 ---- 物理地址 (= BaseOfLoader * 10h)
```



183:       66 ea 80 03 09 00       ljmpw  $0x9,$0x380

反汇编

objdump -D -b binary -m i386 loader.bin > loader-objdump.s

















bochs 教程

https://www.bytekits.com/bochs/bochs-show-mem-cmd.html



[选择 conforming 还是 **non-conforming** ？](http://blog.chinaunix.net/uid-587665-id-2732926.html)

 

分类：

2009-05-05 00:19:51

　　code segment descriptor 的 C 位表示 code segment 是 conforming 还是 **non-conforming** 类型，C = 1 代表 conforming 类型，C = 0 代表 **non-conforming**。conforming 类型可以允许低权限向高权限转移，而不需要通过 call-gate。低权限向高权限的 **non-conforming** 转移则必须通过 call-gate。

显然，使用 **non-conforming** 比使用 conforming 更为安全，因为：

**原因 １、当 far call/jmp 到 code segment 直接跳转时（使用 code segment selector）：**

（１）选择 conforming 类型仅需要 CPL >= DPL 权限就可以了，也就是说：可以直接从低权限向高权限的 code segment 转移，但是 CPL 是不改变的。
　　这样存在着隐患，由于 CPL（当前的 processor 权限）不改变，但是代码却从低权限转到了高权限，导致：若高权限的代码之中使用了高权限的指令，而 CPL 还停留在低权限上，执行这样的指令会导致异常发生，这是其一。
　　其二：高权限的代码使用的是高权限的 stack，但由于 CPL 不变，导致不能发生正确的 stack 切换。

（２）而选择 **non-conforming** 类型需要更严格的权限，需：CPL == DPL 且 RPL <= DPL。
　　很明显，使用 **non-conforming** 类型不能从低权限转向高权限，必须 CPL == DPL，只能要同级权限之中转移。这样就避免了不必要的隐患。



原因 2：当通过 call-gate 来 far call/jmp 到 code segment 时：

（１）选择 conforming 类型的需要权限：(CPL <= DPLg) && (RPL <= DPLg) 并且 (CPL >= DPLs)
（２）选择 **non-conforming** 虽然需要同样的权限：(CPL <= DPLg) && (RPL <= DPLg) 并且 (CPL >= DPLs)。
　　但是，在 **non-conforming** 下仅允许使用 call 指令转移到高权限，不允许使用 jmp 指令转移到高权限代码。
使用 jmp call-gate 导致产生两个问题：
其一：jmp 指令不能正确返回到原调用的代码。由于没有将原 CS:RIP 入栈保存，系统服务例程 ret 时，将会出错。
其二：由于不能正确返回，可能导致不能从高权限代码返回低权限代码，processor 将一直运行在高权限代码中。

　　使用 jmp call-gate 指令到 **non-conforming** 仅允许同级转移，即：CPL <= DPLg && RPL<= DPLg 并且 CPL == DPLs
　　jmp 指令只能同级转移（不能调用系统服务例程），使用 **non-conforming** 类型将不允许 jmp 指令转移到高权限代码，避免隐患。　　

\-------------------------------------------------------------------------------

　　综上所述两个原因，使用 **non-conforming** 类型更符合 x86 / x64 的保护机制。

一个GDT例子

```shell
0000000000000000111111111111111100000000110011111001101100000000
<bochs:15> xp /2wt 0x0000000000090145
[bochs]:
0x0000000000090145 <bogus+       0>:	00000000000000001111111111111111	00000000110011111001101100000000
<bochs:16> info gdt
Global Descriptor Table (base=0x000000000009013d, limit=31):
GDT[0x0000]=??? descriptor hi=0x00000000, lo=0x00000000
GDT[0x0008]=Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
GDT[0x0010]=Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
GDT[0x0018]=Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
```

这个gdt的定义

```assembly
LABEL_DESC_FLAT_C:		Descriptor             0,              0fffffh, DA_CR  | DA_32 | DA_LIMIT_4K			; 0 ~ 4G
```

拆解过，基本是吻合的，没有验证全部属性。



汇编学习网站：

http://www.x86asm.com/#

