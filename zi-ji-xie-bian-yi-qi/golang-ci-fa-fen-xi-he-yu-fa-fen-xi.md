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
 flex -+ -o scanner.cpp token.l 
```

## Makefile

```makefile
CC = g++
OUT = tcc
OBJ = scanner.o parser.o
SCANNER = token.l
PARSER = grammer.y
CFLAGS= -g -O2

build: $(OUT)

run: $(OUT)
	./$(OUT) < test.c > test.log

clean:
	rm -f *.o *.cpp *.hpp y.*  *.cc $(OUT)

$(OUT): $(OBJ)
	$(CC) -g -o $(OUT) $(OBJ) parser.cpp -ll

scanner.cpp: $(SCANNER) parser.cpp
	flex -+ -o scanner.cpp $<

#输出文件：输入文件
#在命令中仍需要写 -o 输出文件
parser.cpp: $(PARSER)
	bison -d -o parser.cpp $<
```

掌握makefile很有必要，例如，在本文件的场景中，若手工执行，需要执行三次命令。

本段笔记的精华是最下面的那两行注释，我花了点时间才通过：

```makefile
$(OUT): $(OBJ)
	$(CC) -g -o $(OUT) $(OBJ) parser.cpp -ll
```

弄明白第24、25行应该怎么写。

## AST设计

怎么设计抽象语法树？抽象语法树中有几种类型的节点？

我能想到一些，例如：函数节点、if节点、while节点。一切需要处理的代码元素，都会表示为一个节点。这些节点实质是继承一个根节点的节点。在面向对象语言中，这种场景很好处理，父类和子类。在C语言中，我使用拥有大量冗余成员的struct来表示。

我不打算空想出所有的节点类型，还是从解析具体的代码着手吧。

## 待解决问题

1. Makefile编写
2. clion格式化代码出问题