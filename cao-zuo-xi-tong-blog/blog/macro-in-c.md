# C语言中的宏

## 什么是宏

抽象的定义不如具体的代码有表现力。先看一段使用宏的代码吧。

```c
// code-A
#include <stdio.h>

#define MIN(a, b) a > b ? b : a

int main(int argc, char **argv)
{
        int a = 5;
        int b = 7;
        int min = MIN(a, b);
        printf("min = %d\n", min);

        return 0;
}
```

执行结果是：

```shell
# 编译
[root@localhost c]# gcc -o macro-demo macro-demo.c
# 运行
[root@localhost c]# ./macro-demo
min = 5
```

定义宏的语句：`#define MIN(a, b) a > b ? b : a`。

从执行结果看，宏`MIN`的作用是，返回`a`和`b`中较小的那个数。

宏的语法格式是：`#define 宏函数（宏参数） 替换体`。

可以回答“什么是宏”这个问题了。宏用一个宏函数指代一个长表达式，使用宏只需使用宏函数即可。

## 宏和函数

```C
// code-B
#include <stdio.h>

int MIN(int a, int b);

int main(int argc, char **argv)
{
        int a = 5;
        int b = 7;
        int min = MIN(a, b);
        printf("min = %d\n", min);

        return 0;
}

int MIN(int a, int b)
{
        return (a > b ? b : a);
}
```

代码`code-B`和`code-A`的执行结果一致，但是，二者调用的`MIN`却并不相同。

在`code-A`中，`MIN`是宏；在`code-B`中，`MIN`是函数。在两份代码中，宏和函数实现的功能虽然相同，本质却不同。

### 占用空间

调用宏三次，编译C代码生成的二进制代码，在调用宏的位置会出现宏代码生成的二进制。在整个二进制代码中，宏代码对应的二进制代码会出现三份。

调用函数三次，在整个二进制代码中，只会出现一份`MIN`函数对应的二进制代码和调用`MIN`函数的二进制代码。

结论显而易见，使用宏会增加最终生成的二进制代码的大小；使用函数，生成的二进制代码相对较小。

### 执行效率

调用宏在预处理阶段直接把宏指向的代码片段插入最终源码文件，在执行过程中，不需要使用压栈的方式向函数传递参数，调用结束后，也不需要通过出栈操作释放栈空间。

调用函数需遵守“函数调用规约”。如果函数有参数，调用函数前，需要先压栈向函数传递参数；调用结束后，需要出栈释放栈空间。

使用函数的效率稍低。在多重嵌套或循环嵌套场景下，如果要提高程序运行效率，宏是更优选择。

## 宏语法

宏函数名要大写。如果写成小写，也不会报错。写成大写，是一种约定。

宏函数名中不允许有空格。这是硬性规定。宏函数名对应的替换体中允许出现空格。

替换体必须写在一行。为了美观，可以使用`\`，然后换行。

```c
// 错误代码。替换体没有写在一行。
#define MIN(a, b) (a > b ?
	b : a)
```

```c
// 正确代码。
#define MIN(a, b) (a > b ? \
	b : a)
```

```c
// 正确代码。
#define MIN(a, b) (a > b ? b : a)
```

替换体用圆括号括起来和不用圆括号括起来是两个不同的宏。

```c
#define NR_PART_PER_HD	4
#define NR_DEV_PER_HD  (NR_PART_PER_HD + 1)
```

```c
#define NR_PART_PER_HD	4
#define NR_DEV_PER_HD2  NR_PART_PER_HD + 1
```

`NR_DEV_PER_HD2`和`NR_DEV_PER_HD`是两个不同的宏。在C代码中使用，可能会计算出完全不同的最终结果。

## 小结

### 2021-07-04 23:23

写完本文。无难点。耗时57分。

以后少写这种文章。收益几乎为零。

### 2021-07-04 23:34

使用 https://www.mdnice.com/ 排版，非常方便。

发文章时需用微信，没控制住自己，看了群聊。

耗时9分钟。