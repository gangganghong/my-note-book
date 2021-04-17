# 实现进程---系统调用

## 代码位置

/home/cg/os/pegasus-os/v23

## 回忆

### v1

系统调用的外在形式是一个函数，用户进程想创建一个弹窗，只需要调用一个系统函数就能打开一个弹窗；用户进程想获得当前时间，只需调用一个函数就能获得系统时间。这些函数就是系统调用。

系统调用，是用户进程请求操作系统做一些事情。

一个函数，内部是一个中断调用，这个中断对应的中断例程处理一些事情。

系统调用的过程是这样的：

1. 用户进程调用系统调用函数。
2. 系统调用函数触发中断。
3. 中断对应的中断例程完成操作。

我的疑问：

1. 系统调用应该只能用汇编写。因为，我只知道触发中断的汇编方法，不知道C语言能写出触发中断的代码。
2. 中断例程完成操作后，怎么把执行结果返回给用户进程？

### v2

1. 在IDT中增加IDT项，确定中断向量号和对应的中断例程函数名。
2. 建立一个用户特权级的系统调用函数，在这个系统调用函数内部触发中断。用汇编完成。我只知道触发中断的汇编方法。
3. 在中断例程内，在中间代码中，完成用户进程要求操作系统做的事情，比如，获取时钟中断发生的次数。

上面都是模板，重点和难点是，系统调用的中断例程要怎么写？

1. 建立快照，和时钟中断例程一致。
2. 中间代码，需要处理中断重入吗？
   1. 在运行中间代码时，能发生的中断有时钟中断和非时钟中断。
      1. 时钟中断。不需要处理。
      2. 非时钟中断。
         1. 系统调用中断。
         2. 非系统调用中断，例如，键盘操作。
         3. 想不明白。先不处理吧。

### v3

怎么写代码？

1. 在kernel_main.c创建变量，`unsigned int ticks`。
2. 在kernel_main函数中把 ticks 初始化成0。
3. 在schedule_process中自增ticks。
4. 在kernel_main.c中增加IDT项：
   1. 中断向量号是90h。
   2. 偏移量，即，中断例程的名称是：sys_call。
   3. 使用 initIDTDesc增加IDT项。
5. 在kernel.asm中增加函数，get_ticks，在它的内部触发90h中断。

#### 疑问

1. ticks是一个全局变量，能直接在用户进程获取吗？如果能，不需要使用系统调用啊，直接获取ticks的值不就行了吗？
2. 试试。
   1. 经过测试，在用户进程中能调用全局变量ticks。如此，保护模式的”保护“体现在哪里？为啥还需要系统调用来获取ticks？
   2. 用于上神的代码也测试了
      1. 也能在用户进程中获取全局变量的值。
      2. 在虚拟机的 /home/cg/yuyuan-os/osfs06/q 能查看运行效果。
      3. 在进程体TestA中，打印ticks。
   3. 搁置这个疑问吧。

## 学习

在用户进程内竟然能直接使用全局变量ticks，保护模式的”保护“体现在哪里？系统调用是否是多余的？直接使用内核方法不就行了？在进程体内不是可以直接使用disp_int这类方法吗？

学习系统调用，非学习那套实现系统调用的代码模板不可。不用模板，仅仅只获取ticks，何必浪费那么多时间写代码？

### v1

#### 流程

1. 建立全局变量ticks，在kernel_main中赋值为0。
2. 增加IDT项
   1. 向量号是：90h
   2. 偏移量是：sys_call
3. 创建函数 get_ticks
   1. 用户进程直接使用这个函数
   2. 在这个函数内部，触发90h中断，传递参数。
4. 创建中断例程sys_call
   1. 建立快照
   2. 中间代码
   3. 恢复进程
   4. 不处理中断重入。没有想明白。
5. sys_call的中间代码
   1. 根据get_ticks传递的参数选择合适的函数。

#### 难点

1. 汇编代码怎么返回值？
   1. get_ticks，把返回值放到eax中。
   2. sys_call，也是把返回值放到eax中。
   3. 中间代码，使用C编写，怎么把返回值放到eax中？

### v2

直接给用户进程使用的get_ticks是汇编函数，实现”返回值“是把返回值放到eax中。

并不是直接使用`mov eax,返回值`把返回值放到eax中。而是把返回值放到中断例程的堆栈中，堆栈中的eax位置。

在中断例程中，在中间代码中，选择sys_get_ticks函数获得ticks。把ticks放到堆栈的eax位置。使用iret把ticks更新到eax中。

sys_get_ticks是C代码写的函数，在汇编中调用它，怎么获得它的返回值？也是直接从eax中获得。

#### get_ticks的实现

1. 触发 sys_call 中断，再传递一个序号eax。
2. 这个序号是包含具体中断处理函数的数组sys_call_table的索引。
3. 第一个函数，[sys_call_table + 4 * 0]；第二个函数，[sys_call_table + 4 * 1]；第N个函数，[sys_call_table + 4 * eax]。

#### 修改进程表的堆栈

把返回值放到要恢复的进程的进程表的堆栈中的eax位置，只需弄明白一点：如何修改？

1. 已知进程表的最低位置，统计一下eax距离这个最低位置多少个字节？假如是N个。
2. eax的位置是：最低位置 + N + 4。

## 写代码

1. 新建syscall.asm，建立get_ticks函数，用汇编语言写
   1. 触发中断
   2. `mov eax, 0`
2. 在kernel_main.c
   1. 建立 `typedef void *system_call`。
   2. 建立系统操作数组`system_call sys_call_table[2]`。
   3. 建立函数sys_get_ticks，用C编写。
3. 在kernel.asm中建立中断例程
   1. 函数，sys_call
   2. 重新编写代码建立快照
   3. 复用restart
   4. 中间代码，重点
      1. 在中间代码前打开中断，在中间代码后关闭中断
      2. 根据get_ticks中传递的序号，即eax的值，选择使用sys_call_table中的函数。
      3. 修改被sys_call挂起的进程的进程表堆栈中的eax位置的数据
4. 增加IDT项
   1. 向量号：90h
   2. 偏移量：sys_get_ticks
   3. 特权级：用户级。

## 调试

```shell
kernel_main.c:237:2: warning: initialization of 'void (*)()' from incompatible pointer type 'int (*)()' [-Wincompatible-pointer-types]
  sys_get_ticks
  ^~~~~~~~~~~~~
kernel_main.c:237:2: note: (near initialization for 'sys_call_table[0]')
kernel_main.c: In function 'init_internal_interrupt':
kernel_main.c:450:25: warning: passing argument 2 of 'InitInterruptDesc' from incompatible pointer type [-Wincompatible-pointer-types]
  InitInterruptDesc(0x90,sys_get_ticks,0x08,0x0E);
                         ^~~~~~~~~~~~~
kernel_main.c:299:47: note: expected 'int_handle' {aka 'void (*)()'} but argument is of type 'int (*)()'
 void InitInterruptDesc(int vec_no, int_handle offset, int privilege, int type)
                                    ~~~~~~~~~~~^~~~~~
```

强制转换sys_call_table中的数据的类型。

门描述符的属性不正确

```shell
interrupt(): soft_int && (gate.dpl < CPL)
00019438884i[CPU0  ] WARNING: HLT instruction with IF=0!
```

可能是系统调用例程中加载ldt等出错了。

```shell
LLDT: selector.ti != 0
```

上面的问题，都解决了。获取的sys_get_ticks的返回值不正确。

已经知道gs的位置是esi，堆栈中eax的位置是`[esi + 11 * 4]`吗？

`mov [esi + 11 * 4], eax`

在堆栈中，从gs的下一个元素到eax，共11个int成员。

### 调试过程

一、C语言报错

1. `typedef void (*func)()`，函数`int get_ticks`不能被赋值为这种类型，需要强制转换类型。例如：

2. ```c
   int sys_get_ticks();
   void sys_call();
   //system_call sys_call_table[1] = {
   int_handle sys_call_table[1] = {
           // warning: initialization of 'void (*)()' from incompatible pointer type 'int (*)()' [-Wincompatible-pointer-types]
           // sys_get_ticks
           // 强制转换类型
           (int_handle) sys_get_ticks
   };
   ```

3. C的编译器的提示信息，不喜欢看，例如

   1. `passing argument 2 of 'InitInterruptDesc' from incompatible pointer type [-Wincompatible-pointer-types]`

二、弄错系统调用流程

实现系统调用，有三个函数：

1. 系统调用中断例程。
   1. 增加IDT项：向量号和偏移量。中断例程就是偏移量。这个IDT项的特权级是1特权级。
2. 提供给用户使用的系统调用函数。例如，get_ticks。在A进程中获取ticks使用get_tickes()。
3. 中断例程中真正完成系统调用请求的函数。例如，sys_get_ticks。它在数组sys_call_table中。

三、汇编函数和C函数的返回值都存储在eax中

1. 我编写的汇编函数，有返回值，只需把返回值存储到eax中。
2. 编写的C函数，只需要使用 return value。在汇编代码中调用这个C函数，能从eax中获取C函数的返回值。

四、中断例程中的堆栈

1. 进入中断例程，CPU选择了TSS的0特权级堆栈，建立快照的时候，也是把数据入栈这个堆栈。
2. 这个堆栈，实际上是被挂起的进程的进程表中的堆栈。
3. 在中间代码，使用内核栈。
4. 恢复进程，又切换到进程表中的堆栈。
5. 进程表的堆栈中，已知gs的地址A，eax的地址是：A + 11 * 4。
6. 系统调用函数的返回值，通过在中断例程中修改运行系统调用函数的进程的进程表的堆栈的eax那个位置的值来传递。

## 遗留问题

没有处理系统调用中断重入问题。暂时想不明白这个过程。

## 总结

回忆的时间太长，浪费时间。

通过写作快速回忆，能回忆多少就回忆多少。回忆不起来迅速搁置。

## 工具

```shell
# 在目录下递归查询
grep -r 'ticks' ./
```

