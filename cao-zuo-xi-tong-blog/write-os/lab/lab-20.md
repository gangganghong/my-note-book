# 实现IPC

## 代码位置

/home/cg/os/pegasus-os/v28

这部分非常难。我已经花了1个小时33分，却没有觉得我看明白了一些东西。

我是怎么做的？

1. 看书，理解细节。
   1. 被assert(la == va)阻碍，我认为，二者不相等。
2. 按代码在书中出现的顺序看代码，看不懂很多细节。
3. 看网络资料OSDV，全英文，很全面，却没用。
4. 不记得具体行为了，又看LDT的初始化。我发现，我已经看不懂我一个月前写的代码了，更别说能讲出来细节。
5. 看不懂，就不想看了，看了凤凰网、博客园。没啥好看的新闻、水文。

这块内容，是个硬骨头。我原以为，我能一天做完。

## 学习

### v1

IPC，进程间通信，是同步，非异步。

要先实现几个调试函数。

#### assert

函数原型：`assert(condition)`。

assert其实不是函数，只是一个宏。

condition成立，什么也不做。condition不成立，打印错误信息，然后，马上停止往下执行，停在此处。

很简单。工作量在assertion_failure函数。

不理解于上神怎么用那么复杂的宏。为什么不写成函数。

#### assertion_failure

##### 魔术常量

这是借用PHP中的术语。

`__FILE__、__BASE_FILE__、__LINE__`。

只有`__BASE_FILE`需要解释。可以在C代码中直接使用。

```shell
Breakpoint 1, assertion_failure (exp=0x0, file=0x804858c "assert-demo.c", base_file=0x804858c "assert-demo.c", line=15)
    at assert-demo.c:21
21		printf("file:%s\nbase_file:%s\nline:%d\n", file, base_file, line);
```

##### C语言的函数的参数加#

```c
#include <stdio.h>

#define ASSERT
#ifdef ASSERT
void assertion_failure(char *exp, char *file, char *base_file, int line);
#define assert(exp)  if (exp) ; \
        else assertion_failure(exp, __FILE__, __BASE_FILE__, __LINE__)
#else
#define assert(exp)
#endif


int main(int argc, char **argv)
{
        int a = 1;
        int b = 3;
        assert(a == b);
        return 0;
}

void assertion_failure(char *exp, char *file, char *base_file, int line)
{
        printf("file:%s\nbase_file:%s\nline:%d\n", file, base_file, line);
        printf("exp:%s\n", exp);

        return;
}
```

`assertion_failure(exp, __FILE__, __BASE_FILE__, __LINE__)`，exp是`char *`类型，实参不加`#`，并且是`a == b`时报错。

```c
[root@localhost c]# gcc -o assert-demo assert-demo.c -g -m32
assert-demo.c: In function 'main':
assert-demo.c:17:11: warning: passing argument 1 of 'assertion_failure' makes pointer from integer without a cast [-Wint-conversion]
  assert(a == b);
         ~~^~~~
assert-demo.c:7:32: note: in definition of macro 'assert'
         else assertion_failure(exp, __FILE__, __BASE_FILE__, __LINE__)
                                ^~~
assert-demo.c:5:30: note: expected 'char *' but argument is of type 'int'
 void assertion_failure(char *exp, char *file, char *base_file, int line);
```

exp是`char *`类型，实参加`#`，并且是`a == b`时，运行正常。

```c
#include <stdio.h>

#define ASSERT
#ifdef ASSERT
void assertion_failure(char *exp, char *file, char *base_file, int line);
#define assert(exp)  if (exp) ; \
        else assertion_failure(#exp, __FILE__, __BASE_FILE__, __LINE__)
#else
#define assert(exp)
#endif


int main(int argc, char **argv)
{
        int a = 1;
        int b = 3;
        assert(a == b);
        return 0;
}

void assertion_failure(char *exp, char *file, char *base_file, int line)
{
        printf("file:%s\nbase_file:%s\nline:%d\n", file, base_file, line);
        printf("exp:%s\n", exp);

        return;
}
```



```shell
[root@localhost c]# gcc -o assert-demo assert-demo.c -g -m32
[root@localhost c]# ./assert-demo
file:assert-demo.c
base_file:assert-demo.c
line:17
exp:a == b
```

c函数传参时，实参前加`#`传输的是内存地址，不加`#`传输的是值。

> 像这样的小知识点，耗费了29分。
>
> 把于上神的代码抽取出来运行验证。语法而已。只需要知道上面的那个结论就行了。

##### printl

这是一个宏，实质是printf。

###### printf

和v27的printf不同，

1. 调用的系统调用不是write，而是printx。
2. 产生字符串的vsprintf也与v27的不同。

###### vsprintf

与v27的vsprintf大不同。

1. 除支持`%x`外还支持`%c`、`%s`、`%d`等。
2. 最关键，算法改变了，和v27实现`%x`很不同。

> 看到这个变化，我很苦恼！原来，IPC这一章，仅仅是两个调试函数，就包含这么多内容。对于想速成的我，简直是晴天霹雳！

###### printx

第一版计算。

中断重入，无法心算执行过程。笔算一下吧。

1. A进程运行中，k = -1。
2. 发生中断int1，inc k，k = 0。
3. 进入调度程序，k = 0。
4. 在调度未结束前，发生中断int2，k = 0，inc k，k = 1。
5. 因为k = 1，不切换堆栈，沿用中断int2发生时的当前堆栈。
6. 当前堆栈是被中断挂起的那个进程即调度程序的堆栈。
7. 也就是说，中断重入时，不调度进程，恢复执行被中断int2挂起的调度程序。dec k ，k = 0。
8. 调度程序执行结束，回到中断例程，继续执行。dec k，k = -1。
9. 也就是说，在处理一个中断的过程中发生了另一个中断，方法是，直接不处理另一个中断。
10. 接着第8步。中断发生，inc k，k = 0，切换堆栈。
11. 结论：
    1. k = 0时，切换堆栈；k != 0时，不切换堆栈。
    2. 此时的特权级呢？我认为，都是在0特权级。

上面的推测过程，正确性未知。

> 今天，大半时间耗费在理解：根据k_reenter的值判断调用printx的特权级。
>
> 毫无章法地盯着代码看，随意推测。无论推测得多合理，只是自圆其说，最好能运行看看执行过程。
>
> 可是，我不知道怎么去测试，只能推测。

第二版计算。

1. 不会直接调用sys_printx，无论哪个特权级的进程，都会使用系统调用间接调用sys_printx。
2. 并非所有代码都通过进程运行。kernel_main中的所有代码（调用的函数），没有在任何进程中。可以认为，它们的特权级是0。
3. TestA、TaskTTY等，以进程的方式执行。可以认为，它们的特权级是1~3。
4. 只有在restart执行之后，才能使用printx。因为，这个函数调用的out_char依赖TTY。

> 又花了31分钟。看代码，推测。
>
> 要换方法了，去运行代码看结果吧。

第三版计算。

~~调试代码：/home/cg/yuyuan-os/osfs08/a~~

调试代码：/home/cg/yuyuan-os/osfs09/a

> 耗费时间1小时16分。
>
> 我一直在做什么？osfs08的代码不能运行，不能加载内核。我不知道是怎么回事。根据屏幕打印信息，我完全不知道怎么去解决那个问题。
>
> 我做了什么？在loader.asm中断点，在kernel.asm中断点。修改kernel.asm为最小的内核。
>
> 我一直不停地做各种修改：代码、配置、断点。就这样马不停蹄地折腾了1个小时16分。
>
> 我还有一个新发现，我完全看不懂我一个多月前写的汇编代码了，当时烂熟于心的读取软盘扇区的知识，我也忘记了。呜呜呜呜.......。
>
> 还好，/home/cg/yuyuan-os/osfs09/a能正确运行。明天，我用它来调试。

##### spin

用printl打印一个语句，然后停留在一个空的死循环。

这个语句是：spinning in func_name

> 这种场景下的死循环，有专门名称吗？

##### 内联汇编

`__asm__  __volatile__`。

`__asm__ __volatile__("ud2");`，产生一个异常。具体是什么异常，试试就知道了。

#### panic

## 工具

### vim

1. `0`到行首，`$`到行尾。
2. `gg`到文档首行，`G`到文档结尾。
3. `Ctrl`+`f`下一页，`Ctrl`+`b`上一页。
4. `Ctrl`+`u`往上半页，`Ctrl`+`d`往下半页。
5. `w`或`e`光标往后跳一个单词，`b`光标往前跳一个单词。
6. `:98`跳转到第98行。
7. `q:`显示**命令行历史记录**窗口。
8. `!bash_Command`不退出vim暂时返回终端界面执行该命令。
9. `H`将光标移动到屏幕首行，`M`将光标移动到屏幕中间行，`L`将光标移动到屏幕最后一行。

> 接触vim很久了，一直都只使用非常常用的那些键，从未去发现其他的键。例如，在一行移动光标的快捷键。

## 总结

### 2021-04-27 23:23

耗时9个小时08分。收获：

1. C函数的实参前加#，能传递内存地址。
2. assert的实现方法。

障碍：

1. 根据k_reenter判断用户进程的特权级是0还是非0。不理解。
2. 我并没有理解”中断重入“的解决方法。

教训：

1. 解决难题，不要心算，要用笔算。
2. 多进程代码的执行流程，我难以模拟。
3. 不要依赖推测，要通过实际运行代码看执行过程。中断重入的代码，推测得再好，也只是自圆其说，不知道是不是正确的。
4. 不要无脑调试！要分析、思考，然后再调试。
5. 9个多小时一直盯着代码看、凭感觉推测多进程代码的执行、小修改小改动就运行。这样太浪费时间了！

IPC真是块难啃的硬骨头！我原本预计一天就能结束它。没想到，它让我如此焦头烂额。