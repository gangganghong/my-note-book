---
description: 使用flex和bison处理golang代码，生成抽象语法树
---

# golang词法分析和语法分析

用C语言还是CPP？

用CPP吧，工作中用得多。我又没有特别多时间既练习CPP又练习C。

为啥不想用CPP？

简化了，例如对字符串的使用，不利于我掌握原生的C。复杂了，引入了class这类东西，容易写一大堆形式的东西。

CPP和bison、flex结合，有方案吗？

例子：/Users/cg/data/code/study-compiler-java/study/flex-and-bison/cpp/grammar.y

在bison文件中需要加入：

```text
void yyerror (const char *error);
int  yylex ();
```

编译命令：

```text
 bison -d -o parser.cpp grammar.y
 g++ parser.cpp 
```



