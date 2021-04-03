# 操作系统--怎么使用中断

中断发生时，操作系统会为当前的任务建立一个快照，陷入内核，把CPU的控制权交给内核。内核趁这个机会做一些工作，比如调度执行其他任务。这只是中断的作用之一。

使用中断有一套固定的流程，掌握它即可。流程大概如下：

## 初始化8259A

初始化工作是对主从`8259A`的两类端口赋值。这两类端口是：`ICW`和`OCW`。

### ICW

这是初始化命令字端口，一共有四个。

1. `ICW1`，设置是否级联、是否向`ICW4`写入数据。
2. `ICW2`，设置主从`8259A`的初始中断向量号。注意，有一批中断向量号已经在实模式下使用，设置初始中断向量号应该避免覆盖那批中断向量号。
3. `ICW3`，主片使用位图标识几号引脚挂载从片。从片使用低3位识别主片发送的数据的接收方是否是自己。
4. `ICW4`，设置是否主动发送`EOI`。不清楚这点。

### OCW

这是控制命令字端口。目前只需要操作一个。

设置屏蔽哪些端口、放行哪些端口。1表示屏蔽，0表示放行。

## 建立IDT

`IDT`是中断向量表。类似`GDT`，也是内存中的一块区域。但它内部包含的全是"门描述符"。

`IDT`中的每个门描述符的名称是中断向量号，选择子是中断向量号对应的处理中断的代码。

```assembly
; Gate : offset，selector，attr，paramCount
%macro Gate 4
dw	%1 & 0FFFFh
dw	%2 & 0FFFFh
dw	((%3 & 0FFh) << 8) | (%4 & 01Fh)
db	%1 >> 16
%endmacro

[SECTION .idt]
LABEL_IDT:
%rep 128
	Gate	selector32,	SpuriousHandler, attr, paramCount
%endrep
.080h:	Gate	selector32,	UserIntHandler, attr, paramCount

IDTLen	equ		$ - LABEL_IDT
IDTPtr	dw	IDTLen - 1
				dd	0
```

### 疑问

中断向量总数是255个还是256个？

![image-20210402084220244](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/dos/image-20210402084220244.png)



`IDT`中，第一个元素是可以用的，这与`GDT`不同。

```assembly
%rep 128
	Gate	selector32,	offset, attr, paramCount
%endrep
```

表示`IDT`中有128个门描述符都指向相同的目标代码段。从0算起，128个门描述符，紧接着的中断向量号应该是128，正好是`80h`。

## 建立中断代码段

```assembly
_SpuriousHandler:
SpuriousHandler	equ	_SpuriousHandler - $$
	mov al, 'A'
	mov ah, 0Fh
	mov [gs:(80*20+20)*2], ax
	ireted
	
_UserIntHandler:
UserIntHandler	equ	_UserIntHandler - $$
	mov al, 'A'
	mov ah, 0Fh
	mov [gs:(80*20+20)*2], ax
	iretd
```

上面的代码和`IDT`在同一代码段code32。`SpuriousHandler`、`UserIntHandler`是code32内的两个偏移量。

不能完全理解这种写法。不过，可暂且当作一种语法规则去记住就行了。以后有类似功能需要实现，就用这种写法。

当中断向量号是在`0~127`（包括0和127）时，会执行中断处理代码`SpuriousHandler`。

当中断向量号是`080h`时，会执行中断处理代码`UserIntHandler`。

## 调用中断

调用中断的语句很简单，如下：

```assembly
int 00h
int 01h
int 10h
int 80h
```



## 特例--时钟中断

增加一个中断的流程：

1. 在`IDT`中增加一个门描述符，提供目标代码（中断处理程序）的选择子、偏移量等。
2. 编写中断处理代码。
3. 调用中断。

时钟中断很特殊，不需要直接调用就能被调用。

编写时钟中断的完整代码如下：

```assembly
; Gate : offset，selector，attr，paramCount
%macro Gate 4
dw	%1 & 0FFFFh
dw	%2 & 0FFFFh
dw	((%3 & 0FFh) << 8) | (%4 & 01Fh)
db	%1 >> 16
%endmacro

[SECTION .idt]
LABEL_IDT:
%rep 128
	Gate	selector32,	SpuriousHandler, attr, paramCount
%endrep
.080h:	Gate	selector32,	UserIntHandler, attr, paramCount
.81h:		Gate	selector32,	ClockIntHandler, attr, paramCount	

IDTLen	equ		$ - LABEL_IDT
IDTPtr	dw	IDTLen - 1
				dd	0
				
_SpuriousHandler:
SpuriousHandler	equ	_SpuriousHandler - $$
	mov al, 'A'
	mov ah, 0Fh
	mov [gs:(80*20+20)*2], ax
	iretd
	
_UserIntHandler:
UserIntHandler	equ	_UserIntHandler - $$
	mov al, 'A'
	mov ah, 0Fh
	mov [gs:(80*20+20)*2], ax
	iretd
  
_ClockIntHandler:
ClockIntHandler	equ	_ClockIntHandler - $$
	inc [gs:(80*20+20)*2]
	mov al, 20h
	out 20h, al
	iretd
	
int 80h
;在死循环中，时钟中断会以一定的时间间隔发生。
jmp $
```

`[gs:(80*20+20)*2]`这块内存已经有数据了，`inc [gs:(80*20+20)*2]`会增加这块内存中的数据的值，效果是在屏幕上出现不断变换的字母。

## 中断或异常发生时堆栈的变化

没有特权级转移的中断，只涉及被中断过程的堆栈，并且堆栈中没有`ss、esp`。

有特权级转移的中断，涉及被中断过程的堆栈和中断例程的堆栈，但是`ss、esp`等都存储在中断例程的堆栈中。

从中断或异常返回时必须使用`iretd`。这与下文使用`iret`矛盾啊。

`iretd`执行时，`Error Code`不会自动从堆栈中清除，这意味着必须手工从堆栈中清除`iretd`。

中断或异常不会总是产生`Error Code`。

### 疑问

在下面的堆栈示意图中，使用`iretd`，会依次出栈除`Error Code`外的其他元素吧？

### 没有特权级转移

|                         |   高地址   |                       |
| ----------------------: | :--------: | --------------------- |
| 中断发生前esp---------> |            |                       |
|                         |   eflags   |                       |
|                         |     cs     | <----------------堆栈 |
|                         |    eip     |                       |
| 中断发生后esp---------> | Error Code |                       |
|                         |   低地址   |                       |

### 有特权级转移

被中断过程的堆栈

|                             | 高地址 |                       |
| --------------------------: | :----: | --------------------- |
| 中断之前的esp-------------> |        |                       |
|                             |        |                       |
|                             |        | <----------------堆栈 |
|                             |        |                       |
|                             | 低地址 |                       |

中断例程的堆栈

|                                   |   高地址   |                       |
| --------------------------------: | :--------: | --------------------- |
|                                   |     ss     |                       |
|                                   |    esp     |                       |
|                                   |   eflags   |                       |
|                                   |     cs     | <----------------堆栈 |
|                                   |    eip     |                       |
| 进入中断例程后的esp-------------> | Error Code |                       |
|                                   |   低地址   |                       |

## 中断或异常的区别

中断会复位`IF`位，防止中断过程中再产生中断，在中断例程结束时通过`iret`恢复`IF`的值。

异常 不会修改`IF`位。

## 记录

2021-04-02 重看。

1. 已经忘记了八九成。如果不重看，也不写出一个操作系统，花在学这些知识的时间，就会被白白浪费，和2016、2017、2018年的那些时间一样，耗费在了无效行为上。
2. 这篇文章是看书后马上写出来的还是看出后过来很久才写出来的？正确性有几成？