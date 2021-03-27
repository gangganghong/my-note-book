# 从高特权级切换到低特权级

## 理论

### 难度

非常非常让我头疼的一个知识点。

花了非常多时间才勉强弄清楚。如今，几乎全忘记了。

### 其他

#### 几个概念

1. `CPL`。存储在哪里？
2. `RPL`。存储在选择子中。
3. `DPL`。存在在描述符的属性中。
4. 一致性代码段。从一个代码段转移到另一个代码段，
   1. CPL、RPL或DPL不发生改变，仍和转移前的代码段一致。是三者中的哪个？

#### 规则

特权级切换有几个规则，我不记得了。

只有代码段和代码段之间，才能够切换特权级。代码段和数据段之间，只存在访问关系。

下面表达式中，是数值上的比较。特权级的关系和数值的关系相反，数值越高，特权级越低。

数据段，CPL <= DPL && RPL <= DPL。

代码段，CPL = DPL && RPL = DPL。

#### 转移方法

高特权级转移到低特权级，使用`ret`等。

低特权转移到高特权级，使用”门“。

#### 怎么证明转移到了低特权级

##### 我的办法

建立一个描述符T，设置为第3特权级，在这个描述符对应的代码段打印字符'L'。能成功打印出'L'，表示成功了。

问题是，怎么让`ret`执行时，执行T指向的代码？

方法是：在执行`ret`前，分别入栈T和0（cs和ip）。入栈的顺序，可以在写代码时确认。我确实没有记住。这不是考试，没记住不要紧。

有疑问：

1. 需要专门设置栈吗？

2. 使用`ret`还是`iret`？

   1. 使用`retf`。

   2. ```assembly
      push SelectStack_3
      push TOP_OF_STACK3
      push SelectFlatX_3
      push 0
      retf
      ```

   3. 代码就是上面那样。

都通过测试吧。

##### 最终办法

如果我的办法行不通，再看于上神的代码确定最终办法。

## 写代码

代码在：`/home/cg/os/pegasus-os/v9`。

| A    | R    | C    | X    | S    | DPL  | DPL  | P    | AVL  | L    | D/B  | G    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 0    | 0    | 0    | 1    | 1    | 1    | 1    | 1    | 0    | 0    | 1    | 1    |

属性值是：`1100 1111 1000`，转换成十六进制是，`0cf8h`。

使用`jmp SelectFlatX_3:0`直接跳转，出错。

```shell
Booting from 0000:7c00
00014803655e[CPU0  ] check_cs(0x0010): non-conforming code seg descriptor dpl != cpl, dpl=3, cpl=0
00014803655e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x0d)
00014803655e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x08)
00014803655i[CPU0  ] CPU is in protected mode (active)
00014803655i[CPU0  ] CS.mode = 32 bit
00014803655i[CPU0  ] SS.mode = 32 bit
```

特权级为3的代码段的选择子的rpl必须是3，对应的描述符的DPL必须是3，这样才能通过`reft`切换到特权级3的代码段。否则会报错`check_cs(0x0010): non-conforming code seg descriptor dpl != cpl, dpl=3, cpl=0`。

我毫无头绪地想了很久很久，结合我所知道的所有规则，仍然无法解释这个情况。

将选择子的rpl设置成3，将选择子对应的描述符的dpl设置成2，出现下面的错误：

```shell
Next at t=14939268
(0) [0x0000000203ee] 0008:00000000000203ee (unk. ctxt): retf                      ; cb
<bochs:12> s
00014939268e[CPU0  ] check_cs(0x0013): non-conforming code seg descriptor dpl != cpl, dpl=2, cpl=3
00014939268e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x0d)
00014939268e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x08)
00014939268i[CPU0  ] CPU is in protected mode (active)
```



```shell
(0) [0x0000000203ee] 0008:00000000000203ee (unk. ctxt): retf                      ; cb
<bochs:24> sreg
es:0x0038, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
cs:0x0008, dh=0x00cf9b00, dl=0x0000ffff, valid=1
	Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
ss:0x0038, dh=0x00cf9300, dl=0x0000ffff, valid=31
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
ds:0x0038, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
fs:0x0038, dh=0x00cf9300, dl=0x0000ffff, valid=1
	Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
gs:0x0043, dh=0x0000f30b, dl=0x8000ffff, valid=7
	Data segment, base=0x000b8000, limit=0x0000ffff, Read/Write, Accessed
ldtr:0x0000, dh=0x00008200, dl=0x0000ffff, valid=1
tr:0x0000, dh=0x00008b00, dl=0x0000ffff, valid=1
gdtr:base=0x000000000002013f, limit=0x47
idtr:base=0x0000000000000000, limit=0x3ff
```



```shell
<bochs:25> s
00029878537e[CPU0  ] return_protected: ss.rpl != cs.rpl
00029878537e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x0d)
00029878537e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x08)
00029878537i[CPU0  ] CPU is in protected mode (active)
00029878537i[CPU0  ] CS.mode = 32 bit
00029878537i[CPU0  ] SS.mode = 32 bit
00029878537i[CPU0  ] EFER   = 0x00000000
00029878537i[CPU0  ] | EAX=60000a4b  EBX=00000600  ECX=00090002  EDX=00000010
00029878537i[CPU0  ] | ESP=0000ffbe  EBP=00000000  ESI=000e007c  EDI=0000007a
00029878537i[CPU0  ] | IOPL=0 id vip vif ac vm RF nt of df if tf sf zf af PF cf
00029878537i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
00029878537i[CPU0  ] |  CS:0008( 0001| 0|  0) 00000000 ffffffff 1 1
00029878537i[CPU0  ] |  DS:0038( 0007| 0|  0) 00000000 ffffffff 1 1
00029878537i[CPU0  ] |  SS:0038( 0007| 0|  0) 00000000 ffffffff 1 1
00029878537i[CPU0  ] |  ES:0038( 0007| 0|  0) 00000000 ffffffff 1 1
00029878537i[CPU0  ] |  FS:0038( 0007| 0|  0) 00000000 ffffffff 1 1
00029878537i[CPU0  ] |  GS:0043( 0008| 0|  3) 000b8000 0000ffff 0 0
00029878537i[CPU0  ] | EIP=000203ee (000203ee)
00029878537i[CPU0  ] | CR0=0x60000011 CR2=0x00000000
```

`return_protected: ss.rpl != cs.rpl`。`ss.rpl` = 0，`cs.rpl` = 2。



```shell
<bochs:13> s
00014939268e[CPU0  ] return_protected: SS.dpl != cs.rpl
00014939268e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x0d)
00014939268e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x08)
00014939268i[CPU0  ] CPU is in protected mode (active)
00014939268i[CPU0  ] CS.mode = 32 bit
00014939268i[CPU0  ] SS.mode = 32 bit
00014939268i[CPU0  ] EFER   = 0x00000000
00014939268i[CPU0  ] | EAX=60000a4b  EBX=00000600  ECX=00090002  EDX=00000010
00014939268i[CPU0  ] | ESP=0000ffbe  EBP=00000000  ESI=000e007c  EDI=0000007a
00014939268i[CPU0  ] | IOPL=0 id vip vif ac vm RF nt of df if tf sf zf af PF cf
00014939268i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
00014939268i[CPU0  ] |  CS:0008( 0001| 0|  0) 00000000 ffffffff 1 1
00014939268i[CPU0  ] |  DS:0038( 0007| 0|  0) 00000000 ffffffff 1 1
00014939268i[CPU0  ] |  SS:0038( 0007| 0|  0) 00000000 ffffffff 1 1
00014939268i[CPU0  ] |  ES:0038( 0007| 0|  0) 00000000 ffffffff 1 1
00014939268i[CPU0  ] |  FS:0038( 0007| 0|  0) 00000000 ffffffff 1 1
00014939268i[CPU0  ] |  GS:0043( 0008| 0|  3) 000b8000 0000ffff 0 0
00014939268i[CPU0  ] | EIP=000203ee (000203ee)
00014939268i[CPU0  ] | CR0=0x60000011 CR2=0x00000000
```





```shell
(0) [0x0000000203ee] 0008:00000000000203ee (unk. ctxt): retf                      ; cb
<bochs:12> s
00014939264e[CPU0  ] check_cs(0x0011): non-conforming code seg descriptor dpl != cpl, dpl=2, cpl=1
00014939264e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x0d)
00014939264e[CPU0  ] interrupt(): gate descriptor is not valid sys seg (vector=0x08)
00014939264i[CPU0  ] CPU is in protected mode (active)
00014939264i[CPU0  ] CS.mode = 32 bit
00014939264i[CPU0  ] SS.mode = 32 bit
00014939264i[CPU0  ] EFER   = 0x00000000
00014939264i[CPU0  ] | EAX=60000a4b  EBX=00000600  ECX=00090002  EDX=00000010
00014939264i[CPU0  ] | ESP=0000ffbe  EBP=00000000  ESI=000e007c  EDI=0000007a
00014939264i[CPU0  ] | IOPL=0 id vip vif ac vm RF nt of df if tf sf zf af PF cf
00014939264i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
00014939264i[CPU0  ] |  CS:0008( 0001| 0|  0) 00000000 ffffffff 1 1
00014939264i[CPU0  ] |  DS:0038( 0007| 0|  0) 00000000 ffffffff 1 1
00014939264i[CPU0  ] |  SS:0038( 0007| 0|  0) 00000000 ffffffff 1 1
00014939264i[CPU0  ] |  ES:0038( 0007| 0|  0) 00000000 ffffffff 1 1
00014939264i[CPU0  ] |  FS:0038( 0007| 0|  0) 00000000 ffffffff 1 1
00014939264i[CPU0  ] |  GS:0043( 0008| 0|  3) 000b8000 0000ffff 0 0
00014939264i[CPU0  ] | EIP=000203ee (000203ee)
```

上面的`dpl`是描述符的dpl，`cpl`是描述符对应的选择子的`rpl`。这是执行`retf`后的打印信息。

非常重要的一点，`retf`执行后，cpl是出栈的选择子中的rpl。非一致性代码段，要能转移，必须满足：

1. cpl = dpl
2. rpl呢？不清楚。

使用`retf`，cs和ds的特权等级必须一致。

## 总结

最大的时间消耗点是，特权级转移规则。

一开始，面对报错信息，毫无头绪，无法应用书上的那些规则。其实，bochs的报错信息已经说得很明白了。

最大收获是：

1. 不惧怕设置描述符的属性。
2. 使用`retf`时cpl来自目标选择子的rpl。
3. 使用`retf`时，cs和ds的特权级（cpl、rpl、dpl）必须相同。三个是否都必须相同，还有待验证。
4. 非一致性代码段，特权级转移，cpl和dpl必须一致。