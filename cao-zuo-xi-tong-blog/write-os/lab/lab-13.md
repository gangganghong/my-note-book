# 实现进程---多进程

## 代码位置

/home/cg/os/pegasus-os/v21

## 回忆

时钟中断例程分成三部分：

1. 为当前运行中的进程建立快照。
2. 中间代码。
3. 启动下一个要运行的进程。

### 中间代码

选择下一个要运行的进程的进程表。要下一个要运行的进程的进程表用`proc_ready_table`表示，不能再用原来那个存储进程表的数组名表示。

### 启动进程

不需要变化。

### 主要工作

实现多进程，主要工作量不在前面三个环节。

#### 实现进程

怎样实现一个进程？

1. 准备好一个进程体，外在表现形式是一个函数，例如，TestA。
2. 初始化描述符。
   1. TSS。
   2. LDT。
   3. 显存描述符。
3. 初始化进程表。
   1. 设置进程表中的LDT的GDT选择子。
   2. 填充LDT。
   3. 初始化段寄存器。
   4. 初始化通用寄存器。
      1. eip，指代进程体的那个函数名。
      2. esp，进程堆栈，一个数组而已。
      3. eflags:0x1202。IF：1，打开中断；IOPL：1。
4. restart

#### 实现多进程

全局只有一个TSS。

每个进程有一个LDT。

每个进程有一个进程表。

实现单进程处理LDT和进程表的代码放进一个循环中，这个循环遍历的对象是存储进程的数组。

上面这句话，说得不好。我没有找到更好的表述方式。

##### 难点

初始化进程表的eip，在实现单进程时，直接赋值进程体的函数名，例如 TestA。

实现多进程时，不能直接用进程体的函数名赋值，而是通过遍历一个数组。

这个数组的元素是一个结构体。

这个结构体的成员有哪些？

1. 进程体的函数名。数据类型暂时试试用 `typedef void (*funcName)()`。
2. 堆栈长度。
3. 暂时只想到上面这两个。

## 学习



## 写代码

1. 实现三个进程。
2. 准备三个进程体。
   1. TestA。已有。
   2. TestB。
   3. TestC。
3. 进程表 proc_table 扩充为 proc_table[3]。
4. 建立一个结构体，名称task，成员有：
   1. 进程体的函数名，数据类型是 `typedef void (*func)()`。
   2. 进程的堆栈大小。
5. 扩充进程的堆栈，proct_stack[A的堆栈的大小 + B的堆栈的大小 + C的堆栈的大小]。
6. init_prot中处理LDT的代码放入循环中。循环对象是proc_table。
7. 初始化进程表的代码放入循环中。循环对象是proc_table。
8. 建立变量proc_ready_table指向即将运行的进程的进程表。

## 调试

```shell
gcc -g -c -fno-builtin -o kernel_main.o kernel_main.c -m32
kernel_main.c:189:3: error: 'TestA' undeclared here (not in a function); did you mean 'test'?
  {TestA, A_STACK_SIZE},
   ^~~~~
   test
kernel_main.c:189:10: error: 'A_STACK_SIZE' undeclared here (not in a function)
  {TestA, A_STACK_SIZE},
          ^~~~~~~~~~~~
kernel_main.c:190:3: error: 'TestB' undeclared here (not in a function); did you mean 'test'?
  {TestB, B_STACK_SIZE},
   ^~~~~
   test
kernel_main.c:190:10: error: 'B_STACK_SIZE' undeclared here (not in a function)
  {TestB, B_STACK_SIZE},
          ^~~~~~~~~~~~
kernel_main.c:191:3: error: 'TestC' undeclared here (not in a function); did you mean 'test'?
  {TestC, C_STACK_SIZE},
   ^~~~~
   test
kernel_main.c:191:10: error: 'C_STACK_SIZE' undeclared here (not in a function)
  {TestC, C_STACK_SIZE},
          ^~~~~~~~~~~~
kernel_main.c: In function 'init_propt':
kernel_main.c:493:57: error: '(Proc *)&proc_table' is a pointer; did you mean to use '->'?
   int ldt_base = VirAddr2PhyAddr(ds_phy_addr, proc_table.ldts);
                                                         ^
                                                         ->
kernel_main.c:495:13: error: lvalue required as increment operand
   proc_table++;
             ^~
kernel_main.c: In function 'kernel_main':
kernel_main.c:572:36: error: '(Task *)&task_table' is a pointer; did you mean to use '->'?
   proc->s_reg.eip = (int)task_table.func_name;
                                    ^
                                    
                                    ->
kernel_main.c:583:13: error: lvalue required as increment operand
   task_table++;
             ^~
make: *** [Makefile:31: kernel.bin] Error 1                                  
```



```shell
00019188456e[CPU0  ] fetch_raw_descriptor: GDT: index (f37) 1e6 > limit (3ff)
00019298701i[CPU0  ] WARNING: HLT instruction with IF=0!
```



C代码中的全局变量定义无效，有一个非常大的初始值。在单独的C文件中，全局变量有效。只是在操作系统代码中有问题。

打印数据太少（没有打印换行符），出错。

切换进程，失败。

没有头绪，不知道错在哪里。反复修改运行了很久，没进展。

全局变量问题，找到规避的方法，但扔不知道原因。通过gdb断点找到了问题。

```
========================================================================
00238127582i[CPU0  ] CPU is in protected mode (active)
00238127582i[CPU0  ] CS.mode = 32 bit
00238127582i[CPU0  ] SS.mode = 32 bit
00238127582i[CPU0  ] EFER   = 0x00000000
00238127582i[CPU0  ] | EAX=fec00e08  EBX=00031340  ECX=00000000  EDX=00000013
00238127582i[CPU0  ] | ESP=000307e0  EBP=0003087c  ESI=0003110a  EDI=00001f40
00238127582i[CPU0  ] | IOPL=1 id vip vif ac vm rf nt of df if tf SF zf af pf cf
00238127582i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
00238127582i[CPU0  ] |  CS:0008( 0001| 0|  0) 00000000 ffffffff 1 1
00238127582i[CPU0  ] |  DS:000d( 0001| 1|  1) 00000000 ffffffff 1 1
00238127582i[CPU0  ] |  SS:0030( 0006| 0|  0) 00000000 ffffffff 1 1
00238127582i[CPU0  ] |  ES:000d( 0001| 1|  1) 00000000 ffffffff 1 1
00238127582i[CPU0  ] |  FS:000d( 0001| 1|  1) 00000000 ffffffff 1 1
00238127582i[CPU0  ] |  GS:0039( 0007| 0|  1) 000b8000 0000ffff 0 0
00238127582i[CPU0  ] | EIP=000307c0 (000307be)
00238127582i[CPU0  ] | CR0=0x60000011 CR2=0x00000000
00238127582i[CPU0  ] | CR3=0x00000000 CR4=0x00000000
(0).[238127582] [0x0000000307be] 0008:00000000000307be (unk. ctxt): add byte ptr ds:[eax], al ; 0000
00238127582i[CMOS  ] Last time is 1618502000 (Thu Apr 15 08:53:20 2021)
00238127582i[XGUI  ] Exit
00238127582i[SIM   ] quit_sim called with exit code 1

```



```
<bochs:2> q
00315688136i[      ] dbg: Quit
00315688136i[CPU0  ] CPU is in protected mode (active)
00315688136i[CPU0  ] CS.mode = 32 bit
00315688136i[CPU0  ] SS.mode = 32 bit
00315688136i[CPU0  ] EFER   = 0x00000000
00315688136i[CPU0  ] | EAX=00030b30  EBX=8a4444a0  ECX=00000000  EDX=00000035
00315688136i[CPU0  ] | ESP=00033ddc  EBP=00033ddc  ESI=00033df5  EDI=00000140
00315688136i[CPU0  ] | IOPL=1 id vip vif ac vm rf nt of df IF tf sf zf AF pf cf
00315688136i[CPU0  ] | SEG sltr(index|ti|rpl)     base    limit G D
00315688136i[CPU0  ] |  CS:0005( 0000| 1|  1) 00000000 ffffffff 1 1
00315688136i[CPU0  ] |  DS:000d( 0001| 1|  1) 00000000 ffffffff 1 1
00315688136i[CPU0  ] |  SS:000d( 0001| 1|  1) 00000000 ffffffff 1 1
00315688136i[CPU0  ] |  ES:000d( 0001| 1|  1) 00000000 ffffffff 1 1
00315688136i[CPU0  ] |  FS:000d( 0001| 1|  1) 00000000 ffffffff 1 1
00315688136i[CPU0  ] |  GS:0039( 0007| 0|  1) 000b8000 0000ffff 0 0
00315688136i[CPU0  ] | EIP=000304e6 (000304e6)
00315688136i[CPU0  ] | CR0=0x60000011 CR2=0x00000000
00315688136i[CPU0  ] | CR3=0x00000000 CR4=0x00000000
(0).[315688136] [0x0000000304e6] 0005:00000000000304e6 (unk. ctxt): add edi, 0x00000002       ; 83c702
00315688136i[CMOS  ] Last time is 1618528601 (Thu Apr 15 16:16:41 2021)
00315688136i[XGUI  ] Exit
00315688136i[SIM   ] quit_sim called with exit code 0

```



启动进程：

1. 使用的堆栈是进程表。
2. tss.esp0 的值正确吗？
   1. 正确的值应该是什么？进程表堆栈的最高地址 + 4。
   2. 有方法验证码？
      1. 认为设置这个地址的值为特别容易识别的值，例如：8.----->没有错误。
3. 执行到 iretd 前面的断点，通过 esp + n 的方式，
   1. 检查进程表中是否正确存储了寄存器的值：除ss、eflags、esp、eip。
      1. esp的值是什么？是进程的堆栈。
   2. 检查进程表是否存储了其他gs、fs、es、ds等寄存器的值。

时钟中断：

1. esp应该指向进程表堆栈，已经存储了ss、eflags、esp、eip。----->没有错误。



时钟中断压栈的时候，应该出现了问题。有时候正常，有时候不正常。



```
<bochs:10> xp /1wx 0x30:0x34b30
[bochs]:
0x0000000000034b30 <bogus+       0>:	0x00030fda
<bochs:11> xp /1wx 0x30:0x34b34
[bochs]:
0x0000000000034b34 <bogus+       0>:	0x00000005
<bochs:12> xp /1wx 0x30:0x34b38
[bochs]:
0x0000000000034b38 <bogus+       0>:	0x00001206
<bochs:13> xp /1wx 0x30:0x34b3c
[bochs]:
0x0000000000034b3c <bogus+       0>:	0x00033e3c
<bochs:14> xp /1wx 0x30:0x34b40
[bochs]:
0x0000000000034b40 <bogus+       0>:	0x0000000d
<bochs:15> xp /1wx 0x30:0x34b44
[bochs]:
0x0000000000034b44 <bogus+       0>:	0xffff0048

```



在不同的进程表之间切换时，出现问题。

用gdb断点看一次。可是，能看出什么呢？

```shell
(gdb) p *proc_ready_table
$2 = {s_reg = {gs = 57, fs = 13, es = 13, ds = 13, edi = 3677421573, esi = 2299807369, ebp = 1676219998, kernel_esp = 178177,
    ebx = 1354772816, edx = 827375665, ecx = 113600, eax = 1183535187, eip = 200713, cs = 5, eflags = 4610, esp = 212608, ss = 13},
  ldt_selector = 73, ldts = {{seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 186 '\272',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
      seg_base_middle = 0 '\000', seg_attr1 = 178 '\262', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 1}
(gdb) c
Continuing.

Breakpoint 1, schedule_process () at kernel_main.c:677
677	{
(gdb) s
680		counter++;
(gdb) s
681		proc_ready_table = &proc_table[counter%3];
(gdb) s
682		return;
(gdb) p *proc_ready_table
$3 = {s_reg = {gs = 57, fs = 13, es = 13, ds = 13, edi = 4283454216, esi = 1996424310, ebp = 4235061284, kernel_esp = 3088237699,
    ebx = 3677421573, edx = 2299807369, ecx = 4159247966, eax = 243712, eip = 200802, cs = 5, eflags = 4610, esp = 212608, ss = 13},
  ldt_selector = 74, ldts = {{seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 186 '\272',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
      seg_base_middle = 0 '\000', seg_attr1 = 178 '\262', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 2}
(gdb) c
Continuing.

Breakpoint 1, schedule_process () at kernel_main.c:677
677	{
(gdb) s
680		counter++;
(gdb) s
681		proc_ready_table = &proc_table[counter%3];
(gdb) s
682		return;
(gdb) p *proc_ready_table
$4 = {s_reg = {gs = 57, fs = 13, es = 13, ds = 13, edi = 8326, esi = 212536, ebp = 212556, kernel_esp = 215856, ebx = 2299807392,
    edx = 49, ecx = 0, eax = 0, eip = 200679, cs = 5, eflags = 4614, esp = 212540, ss = 13}, ldt_selector = 72, ldts = {{
      seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 187 '\273',
      seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
      seg_base_middle = 0 '\000', seg_attr1 = 179 '\263', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}, pid = 0}
(gdb)
```



LDT 描述符

```shell
505		InitDescriptor(&gdt[7], 0xb8000, 0x0FFFF, 0x0F2);
(gdb) p gdt[9]
$4 = {seg_limit_below = 15, seg_base_below = 19270, seg_base_middle = 3 '\003', seg_attr1 = 130 '\202',
  seg_limit_high_and_attr2 = 0 '\000', seg_base_high = 0 '\000'}
(gdb) p gdt[10]
$5 = {seg_limit_below = 15, seg_base_below = 19362, seg_base_middle = 3 '\003', seg_attr1 = 130 '\202',
  seg_limit_high_and_attr2 = 0 '\000', seg_base_high = 0 '\000'}
(gdb) p gdt[11]
$6 = {seg_limit_below = 15, seg_base_below = 19454, seg_base_middle = 3 '\003', seg_attr1 = 130 '\202',
  seg_limit_high_and_attr2 = 0 '\000', seg_base_high = 0 '\000'}
(gdb) p gdt[8]
$7 = {seg_limit_below = 107, seg_base_below = 17024, seg_base_middle = 3 '\003', seg_attr1 = 137 '\211',
  seg_limit_high_and_attr2 = 0 '\000', seg_base_high = 0 '\000'}
```

```shell
(gdb) p proc->ldts
$12 = {{seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 186 '\272',
    seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
    seg_base_middle = 0 '\000', seg_attr1 = 178 '\262', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}
    
    
(gdb) p proc->ldts
$16 = {{seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 186 '\272',
    seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
    seg_base_middle = 0 '\000', seg_attr1 = 178 '\262', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}
    
(gdb) p proc->ldts
$18 = {{seg_limit_below = 65535, seg_base_below = 0, seg_base_middle = 0 '\000', seg_attr1 = 186 '\272',
    seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}, {seg_limit_below = 65535, seg_base_below = 0,
    seg_base_middle = 0 '\000', seg_attr1 = 178 '\262', seg_limit_high_and_attr2 = 207 '\317', seg_base_high = 0 '\000'}}    
```



进程表

```shell
(gdb) p proc_table
$8 = {{s_reg = {gs = 1996443731, fs = 611778308, es = 2214409960, ds = 3224441540, edi = 2365591739, esi = 2213084286, ebp = 3087889077,
      kernel_esp = 3677421574, ebx = 2299807369, edx = 199956062, ecx = 822084536, eax = 340167131, eip = 3910557321, cs = 28836262,
      eflags = 3224457216, esp = 1354772816, ss = 29081649}, ldt_selector = 21248, ldts = {{seg_limit_below = 35664,
        seg_base_below = 64582, seg_base_middle = 139 '\213', seg_attr1 = 94 '^', seg_limit_high_and_attr2 = 254 '\376',
        seg_base_high = 141 '\215'}, {seg_limit_below = 59518, seg_base_below = 24552, seg_base_middle = 181 '\265',
        seg_attr1 = 131 '\203', seg_limit_high_and_attr2 = 196 '\304', seg_base_high = 4 '\004'}}, pid = 280739889}, {s_reg = {
      gs = 4283454208, fs = 1996424310, es = 4242139172, ds = 3088237699, edi = 3677421573, esi = 2299807369, ebp = 1676219998,
      kernel_esp = 178177, ebx = 1354772816, edx = 827375665, ecx = 113600, eax = 1183535187, eip = 4267609084, cs = 3907550861,
      eflags = 3296965916, esp = 827347716, ss = 113600}, ldt_selector = 20563, ldts = {{seg_limit_below = 18059, seg_base_below = 35836,
        seg_base_middle = 94 '^', seg_attr1 = 254 '\376', seg_limit_high_and_attr2 = 141 '\215', seg_base_high = 126 '~'}, {
        seg_limit_below = 59620, seg_base_below = 46340, seg_base_middle = 131 '\203', seg_attr1 = 196 '\304',
        seg_limit_high_and_attr2 = 4 '\004', seg_base_high = 83 'S'}}, pid = 3677429760}, {s_reg = {gs = 1183535187, fs = 3864955876,
      es = 3907026573, ds = 3296965868, edi = 4283454216, esi = 1996424310, ebp = 4235061284, kernel_esp = 3088237699, ebx = 3677421573,
      edx = 2299807369, ecx = 4159247966, eax = 243712, eip = 1354772816, cs = 4283482161, eflags = 1996488310, esp = 3149935100,
      ss = 1347616769}, ldt_selector = 18059, ldts = {{seg_limit_below = 35836, seg_base_below = 65118, seg_base_middle = 141 '\215',
        seg_attr1 = 126 '~', seg_limit_high_and_attr2 = 228 '\344', seg_base_high = 232 '\350'}, {seg_limit_below = 46250,
        seg_base_below = 50307, seg_base_middle = 4 '\004', seg_attr1 = 83 'S', seg_limit_high_and_attr2 = 80 'P',
        seg_base_high = 255 '\377'}}, pid = 3894703871}}
```



> >PANIC<< I/O APIC read at address 0x0000fec00c02 spans 32-bit boundary !



![image-20210416135230793](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210416135230793.png)





![image-20210416135626509](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210416135626509.png)

不同的进程，打印的却总是第三个进程的数据。原因是，三个进程使用了同一个堆栈。三个进程共用堆栈，出栈入栈过程混乱不堪，结果不可预测，也没有分析的价值。

我模拟一下：

1. A进程运行，入栈A。
2. A进程挂起，B进程运行，入栈B。
3. B进程挂起，C进程运行。
4. C进程中，入栈C。打印C。
5. A运行，出栈B。
6. 总之，堆栈很混乱。具体是怎么回事，也许无法预测。所以，要隔离。

![image-20210416140324482](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210416140324482.png)

为啥能解决问题？

1. 束手无策的时候，对照于上神的代码。
2. 我自己修改了dis_pos，在运行restart前，重新设置dis_pos = 2，而不是沿用清屏前设置的dis_pos = 0。
   1. 若沿用前面的dis_pos，显存将非常快被耗尽。
   2. 不显示A、B进程打印出来的数据。我不明白。
3. 在调度程序前打开中断，在调度程序后关闭中断。
   1. sti，把eflags中的if设置成1。
   2. cli，把eflags中的if设置成0。
   3. 为什么要这样做？
      1. 保存快照和恢复进程，都不允许被打断去处理其他中断。
         1. 保存快照和恢复进程，指令不多，运行不会耗费很多时间。
         2. 保存快照，如果被打断，去处理其他中断，堆栈会发生变化。恢复进程时也是如此。要让堆栈发生变化后再回来时又能继续运行，想想都觉得非常麻烦。
         3. 就像我前几个月调试WebCRT的那个聊天室一样，非常非常耗时、容易出错，也许根本不可能弄正确。
         4. 耗费如此巨大的精力、使用如此复杂又容易出错的流程，只为了在非常非常短的时间响应鼠标等操作，不值得。
         5. 索性，把”快照“和”恢复“设计成原子的，作为一个整体去完成。
      2. 在选择下一个要运行的进程时能处理其他中断。
      3. 如果一直不允许处理中断，键盘、鼠标操作都会无效。
   4. 用bochs调试，使用`r`打印eflags

```shell
<bochs:6> c
00019227697i[CPU0  ] [19227697] Stopped on MAGIC BREAKPOINT
(0) Magic breakpoint
Next at t=19227697
(0) [0x00000003058c] 0008:000000000003058c (unk. ctxt): cli                       ; fa
<bochs:7> r
CPU0:
rax: 00000000_00030000
rbx: 00000000_89144689
rcx: 00000000_00000000
rdx: 00000000_00000031
rsp: 00000000_00033820
rbp: 00000000_00033e5c
rsi: 00000000_00031437
rdi: 00000000_00000016
r8 : 00000000_00000000
r9 : 00000000_00000000
r10: 00000000_00000000
r11: 00000000_00000000
r12: 00000000_00000000
r13: 00000000_00000000
r14: 00000000_00000000
r15: 00000000_00000000
rip: 00000000_0003058c
eflags 0x00001202: id vip vif ac vm rf nt IOPL=1 of df IF tf sf zf af pf cf
<bochs:8> s
Next at t=19227698
(0) [0x00000003058d] 0008:000000000003058d (unk. ctxt): jmp .+11  (0x0003059a)    ; eb0b
<bochs:9> r
CPU0:
rax: 00000000_00030000
rbx: 00000000_89144689
rcx: 00000000_00000000
rdx: 00000000_00000031
rsp: 00000000_00033820
rbp: 00000000_00033e5c
rsi: 00000000_00031437
rdi: 00000000_00000016
r8 : 00000000_00000000
r9 : 00000000_00000000
r10: 00000000_00000000
r11: 00000000_00000000
r12: 00000000_00000000
r13: 00000000_00000000
r14: 00000000_00000000
r15: 00000000_00000000
rip: 00000000_0003058d
eflags 0x00001002: id vip vif ac vm rf nt IOPL=1 of df if tf sf zf af pf cf

```

多个进程运行一段时间后，会出错，是因为，显存已经耗尽。

## 总结

多进程实验，到这里，就应该结束了。

### 时间消耗

写多进程代码，大概花了3个小时（写理论知识 + 写代码）。

调试，花了6个多小时。

其中，大概4个多小时，毫无进展；2个小时，解决了问题。

在这两个小时中，解决问题用了大概20分钟，剩余时间，在解释为什么那样做能解决问题、在验证、在写笔记。

### 问题

#### C语法问题

1. 在C代码中，必须先定义再使用。变量定义，必须在变量使用之前。

2. 进程体的函数名，类型是`typedef void (*func)()`。这条知识，被我记住了。

3. 宏定义。语法和定义常量一样。场景，

   1. 例如，A进程的堆栈大小是A_STACK_SIZE，B进程的堆栈大小是B_STACK_SIZE，
   2. 能用宏描述所有进程的堆栈大小STACK_SIZE 是 `#define STACK_SIZE (A_STACK_SIZE + B_STACK_SIZE)`。

4. 数组名不能递增，指向数组的指针能递增。

   ```c
   #include <stdio.h>
   
   int main(int argc, char **argv)
   {
           int arr[3] = {0,1,2};
   
           printf("arr[0] = %d\n", arr[0]);
   
           int *ptr = arr;
           printf("arr[0] = %d\n", *ptr);
           ptr++;
           printf("arr[0] = %d\n", *ptr);
           return 0;
   }
   
   // 运行结果
   ```

5. 结构体person，指针ptr指向person，

   1. 指针指向，`结构体类型 *ptr = person`。
   2. 结构体获取成员，`person.name`。
   3. 指针获取成员，`ptr->name`。

6. 函数的形参是指针类型，对应的实参应该是一个内存地址，即用`&`获取内存地址。

#### 多进程流程

1. 准备好进程体。进程体的外观是一个函数。
2. 初始化TSS，设置tss.ss0，并且用这个tss的描述符填充GDT。
   1. TSS是一段内存，用全局描述符描述。
   2. 从低特权级向高特权级转移，例如，时钟中断，TSS提供高特权级的堆栈(ss和esp)。
   3. C代码中的gdt是一个长度为128的描述符类型数组。我只是把TSS的描述符填充到GDT的第N个元素。
3. 初始化LDT。
   1. LDT是一段内存，里面是局部描述符。
   2. 每个进程有一个LDT，使用选择子从LDT中查询本进程所使用的内存资源。
   3. 每个LDT在GDT中必须有一个描述符描述它，同时，需要一个GDT选择子。
   4. 一个选择子，从LDT还是GDT中查询描述符，取决于选子子的第2个bit（初始序号是0，TI)是0（GDT选择子）还是1(LDT选择子）。
   5. 把LDT的描述符填充到GDT。
4. 初始进程表。
   1. 遍历保护进程表的数组。
   2. 设置进程表的ldt选择子。
   3. 填充ldt中的两个描述符。
      1. cs。
      2. ds。
      3. 只需修改特权级。
   4. 初始化进程表的段寄存器。
      1. 把全局cs的选择子修改低3位后赋值给cs。
      2. 其他段寄存器的值是：把全局ds的选择子修改低3位为101。
   5. 初始化进程表的通用寄存器。
      1. esp：进程的堆栈。每个进程的堆栈不同。
      2. eip：进程体的函数名。
      3. eflags：0x1202。第2个bit永远是1，IOPL = 1，IF = 1。
         1. IOPL：设置特权级。
         2. IF：是否打开外部中断。`sti、cli、iretd`都能修改IF。
   6. 启动进程：
      1. 把esp指向要运行的进程的进程体。
      2. 加载ldt。
      3. 设置tss.esp0。
      4. 出栈：gs、fs、ds、es。顺序可能不正确。
      5. popad。
      6. iretd。
   7. 中断例程：
      1. 建立当前运行中的进程的快照。
         1. 一系列压栈操作。
         2. pushad。
      2. 打开外部中断，sti。
      3. 把esp指向内核栈。
      4. 调度程序等。
      5. 关闭外部中，cli。
   8. 把esp指向要运行的进程的实体。
   9. 启动进程。

建立快照和启动进程都是原子操作。

时钟中断发生后，外部中断是不是被自动关闭了？

实现字符跳动效果的语句是：`inc [gs:0]`，让屏幕左上角第一个字符跳动。

必须向20h端口写入EOI才能让时钟中断多次发生。

#### 遇到的问题

也就是非预期效果。

1. 并没有看到三个进程交替运行的效果。
2. 进程体内部打印换行符就出错。原因未知，可能是其他问题导致的。
3. 运行一段时间后，就出错。原因是，显存耗尽。

重点说第1个效果。

可能是因为在清屏前设置dis_pos = 0，清屏后，dis_pos增加了许多。进程打印字符仍然继承了之前的dis_pos。未验证。

全部显示第3个进程运行的效果。

原因是3个进程使用同一个堆栈。起初，我不知道怎么在遍历中给每个进程表的esp赋予正确的值，让3个进程表使用了同一个堆栈。

3个进程表使用同一个堆栈，在不可预测的进程运行顺序中，A进程入栈的元素可能被B或C出栈。这非常混乱。我无法模拟这种混乱的执行过程，模拟这种条件下的堆栈操作没有任何价值。

使用一个进程的目的是隔离资源，当然不应该共用堆栈。

给3个进程分配不同的堆栈后，能看到3个进程交替打印的效果了。

### 调试过程

1. 无脑调试了一阵子。
2. 边散步边思考如何调试。心算能力弱。在散步时思考只能是辅助方式，找找方向还行。
3. 检查进程调度是否正确选择了要运行的进程。
   1. 发现进程表中的ldt的属性被修改。原因未知。差点一直沿着这个线索继续追踪下去。
   2. C语言中的全局变量问题。在函数外创建的变量赋予的值，在函数内无效。在函数内赋值，在本函数和其他函数内才有效。
   3. 无效，是指，赋值为0，打印的时候，却是其他很奇怪的值。
   4. 可我在单独的C代码中，在函数外创建的变量赋予的值，在函数内有效。
   5. 比较指针变量所表示的内存地址来重置proc_ready_table。
4. 通过给tss设置很特别的选择子，例如999，来查看启动进程时是否成功获取了tss的选择子。
5. 查看启动进程、建立快照的过程中的堆栈，发现：
   1. 大部分时间正常。
   2. 多进程切换过程中，有时候不正常。
   3. 不知道原因。
6. 万分沮丧的时候，我决定，看于上神的代码。
   1. 看到了关闭中断、打开中断。我遗漏了这些。补上这些后，仍然看不到预期效果。
   2. 清屏后再设置dis_pos = 0，看到屏幕上前面三个是交替打印的，后面仍然全部是第3个进程的。
   3. 突然怀疑，是三个进程共用一个堆栈造成的。修改后，看到了久违的预期效果。
7. 看于上神的代码并没有直接解决我的问题。如果不看他的代码，我不可能想到我的中断有问题。