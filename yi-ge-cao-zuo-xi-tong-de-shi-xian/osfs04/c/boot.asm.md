# boot.asm

## 代码

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

BaseOfLoader		equ	09000h	; LOADER.BIN 被加载到的位置 ----  段地址
OffsetOfLoader		equ	0100h	; LOADER.BIN 被加载到的位置 ---- 偏移地址

RootDirSectors		equ	14	; 根目录占用空间
SectorNoOfRootDirectory	equ	19	; Root Directory 的第一个扇区号
SectorNoOfFAT1		equ	1	; FAT1 的第一个扇区号 = BPB_RsvdSecCnt
DeltaSectorNo		equ	17	; DeltaSectorNo = BPB_RsvdSecCnt + (BPB_NumFATs * FATSz) - 2
					; 文件的开始Sector号 = DirEntry中的开始Sector号 + 根目录占用Sector数目 + DeltaSectorNo
;================================================================================================

	jmp short LABEL_START		; Start to boot.
	nop				; 这个 nop 不可少

	; 下面是 FAT12 磁盘的头
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

LABEL_START:	
	mov	ax, cs
	mov	ds, ax
	mov	es, ax
	mov	ss, ax
	mov	sp, BaseOfStack

	; 清屏
	; 这是个参数固定的BIOS中断，找本权威的书看看。
	; 如何单独运行这段程序？
	mov	ax, 0600h		; AH = 6,  AL = 0h
	mov	bx, 0700h		; 黑底白字(BL = 07h)
	mov	cx, 0			; 左上角: (0, 0)
	mov	dx, 0184fh		; 右下角: (80, 50)
	int	10h			; int 10h

	mov	dh, 0			; "Booting  "
	call	DispStr			; 显示字符串
	
	xor	ah, ah	; ┓
	xor	dl, dl	; ┣ 软驱复位
	int	13h	; ┛
	
; 下面在 A 盘的根目录寻找 LOADER.BIN
; SectorNoOfRootDirectory	equ	19	; Root Directory 的第一个扇区号
	mov	word [wSectorNo], SectorNoOfRootDirectory
LABEL_SEARCH_IN_ROOT_DIR_BEGIN:
	; RootDirSectors		equ	14	; 根目录占用空间
	; wRootDirSizeForLoop	dw	RootDirSectors	; Root Directory 占用的扇区数, 在循环中会递减至零.
	; 遍历根目录的循环次数，是根目录区根目录条目的数量，每遍历一个条目，此值减去1。
	cmp	word [wRootDirSizeForLoop], 0	; ┓
	; 当 [wRootDirSizeForLoop] 等于0时，说明并没有找到目标文件，跳转到 LABEL_NO_LOADERBIN
	jz	LABEL_NO_LOADERBIN		; ┣ 判断根目录区是不是已经读完
	dec	word [wRootDirSizeForLoop]	; ┛ 如果读完表示没有找到 LOADER.BIN
	mov	ax, BaseOfLoader
	mov	es, ax			; es <- BaseOfLoader
	mov	bx, OffsetOfLoader	; bx <- OffsetOfLoader	于是, es:bx = BaseOfLoader:OffsetOfLoader
	; wSectorNo		dw	0		; 要读取的扇区号，是相对于根目录区的扇区号
	mov	ax, [wSectorNo]	; ax <- Root Directory 中的某 Sector 号
	; 读取一个扇区，重点，难点
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
	; 条目的开始是32的倍数，即低5位全是0，（二进制形式）
	and	di, 0FFE0h		; di -> 当前条目的开始
	; DIR_FstClus 的初始位置是 FATEntry的第26个字节。01AH = 26。
	; 没有很快理解这里，一是因为把"add"看成了"and"，二是因为没有结合FATEntry的数据结构来理解。
	add	di, 01Ah		; di -> 首 Sector
	mov	cx, word [es:di]
	push	cx			; 保存此 Sector 在 FAT 中的序号
	add	cx, ax
	add	cx, DeltaSectorNo	; cl <- LOADER.BIN的起始扇区号(0-based)
	mov	ax, BaseOfLoader
	mov	es, ax			; es <- BaseOfLoader
	mov	bx, OffsetOfLoader	; bx <- OffsetOfLoader
	mov	ax, cx			; ax <- Sector 号

LABEL_GOON_LOADING_FILE:
	push	ax			; `.
	push	bx			;  |
	mov	ah, 0Eh			;  | 每读一个扇区就在 "Booting  " 后面
	mov	al, '.'			;  | 打一个点, 形成这样的效果:
	mov	bl, 0Fh			;  | Booting ......
	int	10h			;  |
	pop	bx			;  |
	pop	ax			; /

	mov	cl, 1
	call	ReadSector
	; ax 是上文计算出来的cx。mov	cx, word [es:di]
	; push	cx			; 保存此 Sector 在 FAT 中的序号。此Sector是文件在数据区的第一个扇区的编号。
	; 汇编的难理解之处，ax的值是什么？要在整个代码中寻找与此pop对应的push。寄存器类似全局变量。
	pop	ax			; 取出此 Sector 在 FAT 中的序号
	call	GetFATEntry
	; 若FAT项的序号是0x0FFF，那么，目前读取到的数据区域的这个扇区是本文件的最后一个扇区。
	cmp	ax, 0FFFh
	jz	LABEL_FILE_LOADED
	push	ax			; 保存文件的下一个 Sector 在 FAT 中的序号
	; 计算文件的下一个簇号（扇区号）相对于软盘的扇区号 start
	; ax 是文件的下一个扇区相对于数据区的扇区号。
	; 要转化成相对于软盘的扇区号，需要加上，引导扇区 + 两个FAT所占用的扇区 + 根目录占用的扇区 （A)。
	; 由于数据区的扇区的初始编号是2，所以，需要将 ax - 2 (B)。
	; 最终结果 = A + B = ax - 2 + RootDirSectors + 19 = ax + RootDirSectors + 17。
	mov	dx, RootDirSectors
	add	ax, dx
	; DeltaSectorNo = 19 - 2。
	; 因为数据区的开始簇号（扇区号）是2，簇号为2的簇（扇区）是数据区的第个扇区。
	; 根据代码，int 13h 需要的扇区号，是当前柱面、当前磁头的扇区号，是从引导扇区开始统计的扇区号，而不是数据区的扇区号。
	add	ax, DeltaSectorNo
	; 计算文件的下一个簇号（扇区号）相对于软盘的扇区号 end
	add	bx, [BPB_BytsPerSec]			; [es:bx]才能让数据连续复制到正确的位置。
	jmp	LABEL_GOON_LOADING_FILE
LABEL_FILE_LOADED:

	mov	dh, 1			; "Ready."
	call	DispStr			; 显示字符串

; *****************************************************************************************************
	jmp	BaseOfLoader:OffsetOfLoader	; 这一句正式跳转到已加载到内
						; 存中的 LOADER.BIN 的开始处，
						; 开始执行 LOADER.BIN 的代码。
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
	; x是相对于软盘的扇区号，不是相对于数据区或根目录区的扇区号。
	; x/每磁道扇区数，商是磁道数（总计有多少个磁道），余数是不足一个磁道的扇区数，即在某个磁道
	; 内的扇区偏移量。
	; 柱面号。两个磁道组成一个柱面，磁道数是4，柱面号是2；磁道数是3，柱面号是1（第0个磁道、第1个磁道组成第1个柱面）。
	; 磁头号。磁头号只有0和1两种。第0个磁道在磁头号为0的盘面，第1个磁道在磁头号为1的盘面，第2个磁道在磁头号为0的盘面，第3个
	; 磁道在磁头号为1的盘面。前面磁道的编号（第0个、第1个）是若干个扇区转换成的若干磁道的原始排列（不是先排满柱面上的所有磁道
	; 号相同的磁道的那种排列方式）。这么理解，A个扇区相当于B个磁道，B个磁道用2个一对的方式用磁道号编号。唉，问题虽简单，想
	; 说清楚却不容易啊。B个磁道按每2个的方式分布到2个盘面上（第0个在盘面0，第1个在盘面1，第2个在盘面0，第3个在盘面1）。显然，
	; 磁道的序号为奇数分布在盘面1，序号为偶数分布在磁道0。判断奇数偶数的方式是 y & 1。
	; z是偏移量，第1个扇区是编号为0的引导扇区，有效起始扇区号 = z + 1。后面再详细解释。
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
	mov	bl, [BPB_SecPerTrk]	; bl: 除数，被除数是ax。
	div	bl			; y 在 al 中, z 在 ah 中
	inc	ah			; z ++
	mov	cl, ah			; cl <- 起始扇区号
	mov	dh, al			; dh <- y
	shr	al, 1			; y >> 1 (其实是 y/BPB_NumHeads, 这里BPB_NumHeads=2)，柱面号
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
	; 上面的push操作，是由于寄存器中的值会在下文被修改。
	mov	ax, BaseOfLoader; `.
	; 段地址增加了0100h，那么，内存空间增加了 0100h > 4 ，因为内存的计算方式是：段地址 * 16 + 偏移量
	sub	ax, 0100h	;  | 在 BaseOfLoader 后面留出 4K 空间用于存放 FAT
	mov	es, ax		; /
	; ax 为啥变成了 00000003 ？ 之前存储的cx的值是 90000003
	pop	ax
	; 奇偶标志，0是偶数，1是奇数。此标志命名，不适合我的理解方式。
	mov	byte [bOdd], 0
	; 难点 start
	; 这几行代码的功能设置 [bOdd] 的值。当编号为ax的FATEntry的偏移量是整数个字节时，[bOdd] 是0；否则，是1。
	; 怎么判断编号是ax的FATEntry的偏移量是不是整数个字节呢？
	; 编号就是以FATEntry所占用大小的偏移量，初始值是0。
	; 一个FATEntry占用1.5个字节，编号为X的FATEntry占用 X * 1.5 个字节，编号为X的FATEntry的字节偏移量是 X * 1.5 个字节。
	; 只需判断 X * 1.5 的结果是不是整数。
	; 汇编中不支持 M： X * 1.5 ，转换为等价运算Y： 先乘以3，再除以2。
	; 当Y的余数是0时，M 是整数；否则，M不是整数。
	mov	bx, 3
	mul	bx			; dx:ax = ax * 3
	mov	bx, 2
	div	bx			; dx:ax / 2  ==>  ax <- 商, dx <- 余数
	; 难点 end
	; 当 dx（余数）是0时，FAT项的偏移量是整数个字节。
	cmp	dx, 0
	; 跳转到整数个字节的偏移量的处理逻辑
	jz	LABEL_EVEN
	mov	byte [bOdd], 1
LABEL_EVEN:;偶数
	; 现在 ax 中是 FATEntry 在 FAT 中的偏移量,下面来
	; 计算 FATEntry 在哪个扇区中(FAT占用不止一个扇区)
	; dx 重设为0
	xor	dx, dx
  ; bx 的值是一个扇区包含的字节数
	mov	bx, [BPB_BytsPerSec]
	; ax / bx，ax 是字节数
	div	bx ; dx:ax / BPB_BytsPerSec
		   ;  ax <- 商 (FATEntry 所在的扇区相对于 FAT 的扇区号)
		   ;  dx <- 余数 (FATEntry 在扇区内的偏移)
	push	dx
	mov	bx, 0 ; bx <- 0 于是, es:bx = (BaseOfLoader - 100):00
	; SectorNoOfFAT1		equ	1	; FAT1 的第一个扇区号 = BPB_RsvdSecCnt
	; 软盘的第一个扇区号是0(引导扇区)，所以FAT1的第一个扇区号是1。
	; ax 是相对于FAT1的扇区偏移量，ax所表示的扇区相对于整个软盘的扇区号是 ax + SectorNoOfFAT1。
	; 前面的理解 “软盘的第一个扇区号是0(引导扇区)，所以FAT1的第一个扇区号是1”， 不怎么正确。
	; 这样理解更好：FAT项的偏移量是ax，ax是相对于FAT1的偏移量，而FAT1相对于软盘的偏移量是 SectorNoOfFAT1，
	; 所以，FAT项相对于整个软盘的偏移量是 SectorNoOfFAT1 + ax。
	add	ax, SectorNoOfFAT1 ; 此句之后的 ax 就是 FATEntry 所在的扇区号
	; 由于FAT项的长度是12个bit，所以，一个FAT项有可能跨越两个扇区。（ 512 / 12)有余数，即一个扇区可能包含一个FAT项的一部分。
	mov	cl, 2
	call	ReadSector ; 读取 FATEntry 所在的扇区, 一次读两个, 避免在边界
			   ; 发生错误, 因为一个 FATEntry 可能跨越两个扇区
	pop	dx
	add	bx, dx
	; 被赋值给ax的是什么？
	; es 是上面计算出来的ax，是段地址。
	; ReadSector 把包含FATEntry的两个扇区读取到了es:bx位置。bx又被赋值为dx（dx是编号为X的FATEntry）在扇区内的偏移量。
	; 这两个扇区又被完整地读取到了内存中（段地址是es），所以，根据软盘中的数据布局计算出来的偏移量在这段内存中同样适用。
	; ax 是内存中的值，不是内存地址。在具体场景中，是FAT项的值，即文件在数据区的下一个簇号。
	; GetFATEntry，参数是文件中某块数据的簇号，结果是下一块数据的簇号。
	mov	ax, [es:bx]
	cmp	byte [bOdd], 1
	; 不是奇数时跳转
	jnz	LABEL_EVEN_2
	; 若FAT项占用的不是整数个字节，ax偏移量却是整数个字节（上一个FAT项的终点-4），从ax开始的16个字节中
	; 有4个字节是上一个FAT项的组成部分，故当前FAT项的内存区域是这个16个字节的高12位。
	; 实在觉得不好解释，那就画图，用具体的例子来解释吧。
	; 反正，上面的解释，是正确的。
	shr	ax, 4
LABEL_EVEN_2:
	; 若偏移量是整数个字节，获取ax的第0~11位，刚好是一个FAT项
	and	ax, 0FFFh

LABEL_GET_FAT_ENRY_OK:

	pop	bx
	pop	es
	ret
;----------------------------------------------------------------------------

times 	510-($-$$)	db	0	; 填充剩下的空间，使生成的二进制代码恰好为512字节
dw 	0xaa55				; 结束标志
```



boot.asm 包含 BPB头等信息，结尾是魔数 0xaa55，一共512字节。

耗费了1小时45分钟。

## 资料

https://www.cnblogs.com/mlzrq/category/1371290.html?page=2

https://yangwenbo.com/articles/writing-x86-pc-bootloader-with-free-software-2.html