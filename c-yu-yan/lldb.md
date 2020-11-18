---
description: lldb是调试C/C++的工具，和GDB的功能类似。
---

# lldb

## lldb

在我的macbook电脑上，使用gdb报错：

```text
Type "apropos word" to search for commands related to "word"...
BFD: /Users/cg/data/code/study-compiler-java/study/flex-and-bison/flex/golang/tcc: unknown load command 0x32
BFD: /Users/cg/data/code/study-compiler-java/study/flex-and-bison/flex/golang/tcc: unknown load command 0x32
"/Users/cg/data/code/study-compiler-java/study/flex-and-bison/flex/golang/tcc": not in executable format: file format not recognized
```

百度到的解决方法很少且雷同，都不管用。所以，我选择用lldb。

在我的macbook上，lldb命令直接可用。

### 使用方法

```text
# 使用 -g ，调试的时候显示C代码而不是汇编代码。
gcc -g -o hello hello.c
lldb hello
# 在dump函数设置断点
b dump
# 在createStr函数设置断点
b createStr
# 开始执行程序
run
# 按程序要求提供输入数据
if(true)ab=5;
# 执行程序直至遇到断点
c
# 打印变量值
p *root
# 打印变量的内存地址
p root
# 退出调试
exit
```

### 资料

\[LLDB十分钟快速教程\]\([https://zhuanlan.zhihu.com/p/106415182](https://zhuanlan.zhihu.com/p/106415182)\)

## C语言字符串的动态操作

[https://blog.csdn.net/qq\_23944945/article/details/83421176](https://blog.csdn.net/qq_23944945/article/details/83421176)

很不错的资料，有空整理学习一下

