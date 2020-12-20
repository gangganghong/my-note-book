---
description: 使用flex和bison处理golang代码，生成抽象语法树
---

# golang词法分析和语法分析

## golang词法分析和语法分析

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

### Makefile

```text
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

```text
$(OUT): $(OBJ)
    $(CC) -g -o $(OUT) $(OBJ) parser.cpp -ll
```

弄明白第24、25行应该怎么写。

### Plan9

官方资料

[https://golang.org/doc/asm\#amd64](https://golang.org/doc/asm#amd64)

go 博客

[https://mzh.io/](https://mzh.io/)

[https://xargin.com/plan9-assembly/](https://xargin.com/plan9-assembly/)

go编译器介绍

[https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-machinecode/](https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-machinecode/)

-gcflags '\[pattern=\]arg list' arguments to pass on each go tool compile invocation.

### 链接时进行的处理

#### 合并节

![image-20201219170207547](https://github.com/gangganghong/my-note-book/tree/44f46d191712a6ab484778eaa819a763c922d9bd/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201219170207547.png)

把多个文件中的`.text`、`.data`等归类并放入同一个文件中的相应类别。

很简单的知识，只需要仔细看一眼就明白了。

#### 重定位

![image-20201219170552097](https://github.com/gangganghong/my-note-book/tree/44f46d191712a6ab484778eaa819a763c922d9bd/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201219170552097.png)

把虚拟内存地址换成实际内存地址。

#### 符号消解

英语叫 Symbol resolution。

未定义符号 undefined symbol。

### AST设计

怎么设计抽象语法树？抽象语法树中有几种类型的节点？

我能想到一些，例如：函数节点、if节点、while节点。一切需要处理的代码元素，都会表示为一个节点。这些节点实质是继承一个根节点的节点。在面向对象语言中，这种场景很好处理，父类和子类。在C语言中，我使用拥有大量冗余成员的struct来表示。

我不打算空想出所有的节点类型，还是从解析具体的代码着手吧。

tokens.cpp:929:7: error: member reference type 'std::ostream _' \(aka 'basic\_ostream_ '\) is a pointer; did you m

### 待解决问题

1. Makefile编写
2. clion格式化代码出问题
3. /Users/cg/data/code/study-compiler-java/src/Makefile

   这个文件，多次调整，都不能正确使用make，我不知道原因。浪费了快2个小时，我都没兴趣去做正事了。下次再遇到这种细枝末节的问题，花了半个小时后还不能解决，迅速跳过，我暂时用shell脚本代替了make。调试能力仍没有提高啊，我仍旧随便改改就尝试一下，试了很多同质方法，事后我也不记得我试了什么方法。

4. golang代码能够翻译成gas汇编代码吗？ 1. 找了网络资料，没找到，只有go代码的编译过程。 2. go打印出来的汇编代码，非常非常长。看不出规律，也不会写。 3. 浪费了很多时间，寸步难行。

### Plan9

[https://mzh.io/about/](https://mzh.io/about/)

plan9 assembly 完全解析

[https://github.com/cch123/golang-notes/blob/master/assembly.md\#plan9-assembly-%E5%AE%8C%E5%85%A8%E8%A7%A3%E6%9E%90](https://github.com/cch123/golang-notes/blob/master/assembly.md#plan9-assembly-%E5%AE%8C%E5%85%A8%E8%A7%A3%E6%9E%90)

### 其他汇编

汇编语言入门：CALL和RET指令

### 好博客

教育对人的改变有多大？

[http://xiaohanyu.me/posts/2017-02-13-about-education/](http://xiaohanyu.me/posts/2017-02-13-about-education/)

## _**ret指令和retf指令\**_

_**`ret`指令用栈中的数据，修改IP的内容，从而实现近转移\**_ _**`retf`指令用栈的数据，修改CS和IP的内容，从而实现远转移\**_

_**CPU执行`ret`指令时，相当于进行：\**_

```text
pop IP1
```

_**CPU执行`retf`指令时，相当于进行：\**_

```text
pop IP
pop CS
```

### golang 编译

![image-20201219155100999](https://github.com/gangganghong/my-note-book/tree/44f46d191712a6ab484778eaa819a763c922d9bd/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201219155100999.png)

![image-20201219155150198](https://github.com/gangganghong/my-note-book/tree/44f46d191712a6ab484778eaa819a763c922d9bd/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201219155150198.png)

查看go命令执行过程

go build -n main\_t.go

[https://blog.csdn.net/qq\_33339479/article/details/105206548?utm\_medium=distribute.pc\_relevant.none-task-blog-BlogCommendFromBaidu-1.control&depth\_1-utm\_source=distribute.pc\_relevant.none-task-blog-BlogCommendFromBaidu-1.control](https://blog.csdn.net/qq_33339479/article/details/105206548?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.control)

go编译器相关

/usr/local/go/src/cmd/go/main.go

/usr/local/go/src/cmd/link/main.go

golang代码能编译成gas等非plan9汇编吗？

目前，我认为是不能，比如，汇编中的call fmt.Printf，如何处理？

在C代码中，call printf，是如何被处理的？

找到这个问题的答案，golang编译器才可以继续写下去。

当然，我也可以先做完词法分析、语法分析、生成AST甚至中间代码。

```text
chugangdeMacBook-Pro:demo cg$ go build -x hello.go
WORK=/var/folders/b3/mjqwy7314nx3n66p9c6tk7100000gn/T/go-build783965543
mkdir -p $WORK/b001/
cat >$WORK/b001/importcfg.link << 'EOF' # internal
packagefile command-line-arguments=/Users/cg/Library/Caches/go-build/17/17a37acb856cb4368bc6d6e5cf5f85de4b0e0a0d327c6503f2cc428067527bf9-d
packagefile fmt=/usr/local/go/pkg/darwin_amd64/fmt.a
packagefile runtime=/usr/local/go/pkg/darwin_amd64/runtime.a
packagefile errors=/usr/local/go/pkg/darwin_amd64/errors.a
packagefile internal/fmtsort=/usr/local/go/pkg/darwin_amd64/internal/fmtsort.a
packagefile io=/usr/local/go/pkg/darwin_amd64/io.a
packagefile math=/usr/local/go/pkg/darwin_amd64/math.a
packagefile os=/usr/local/go/pkg/darwin_amd64/os.a
packagefile reflect=/usr/local/go/pkg/darwin_amd64/reflect.a
packagefile strconv=/usr/local/go/pkg/darwin_amd64/strconv.a
packagefile sync=/usr/local/go/pkg/darwin_amd64/sync.a
packagefile unicode/utf8=/usr/local/go/pkg/darwin_amd64/unicode/utf8.a
packagefile internal/bytealg=/usr/local/go/pkg/darwin_amd64/internal/bytealg.a
packagefile internal/cpu=/usr/local/go/pkg/darwin_amd64/internal/cpu.a
packagefile runtime/internal/atomic=/usr/local/go/pkg/darwin_amd64/runtime/internal/atomic.a
packagefile runtime/internal/math=/usr/local/go/pkg/darwin_amd64/runtime/internal/math.a
packagefile runtime/internal/sys=/usr/local/go/pkg/darwin_amd64/runtime/internal/sys.a
packagefile internal/reflectlite=/usr/local/go/pkg/darwin_amd64/internal/reflectlite.a
packagefile sort=/usr/local/go/pkg/darwin_amd64/sort.a
packagefile math/bits=/usr/local/go/pkg/darwin_amd64/math/bits.a
packagefile internal/oserror=/usr/local/go/pkg/darwin_amd64/internal/oserror.a
packagefile internal/poll=/usr/local/go/pkg/darwin_amd64/internal/poll.a
packagefile internal/syscall/unix=/usr/local/go/pkg/darwin_amd64/internal/syscall/unix.a
packagefile internal/testlog=/usr/local/go/pkg/darwin_amd64/internal/testlog.a
packagefile sync/atomic=/usr/local/go/pkg/darwin_amd64/sync/atomic.a
packagefile syscall=/usr/local/go/pkg/darwin_amd64/syscall.a
packagefile time=/usr/local/go/pkg/darwin_amd64/time.a
packagefile unicode=/usr/local/go/pkg/darwin_amd64/unicode.a
packagefile internal/race=/usr/local/go/pkg/darwin_amd64/internal/race.a
EOF
mkdir -p $WORK/b001/exe/
cd .
/usr/local/go/pkg/tool/darwin_amd64/link -o $WORK/b001/exe/a.out -importcfg $WORK/b001/importcfg.link -buildmode=exe -buildid=LHaX9c3GvkKRZADW4SpA/v_LvXpN-MR1471BbfrBR/E-RANQJaIeMqVRQRod6z/LHaX9c3GvkKRZADW4SpA -extld=clang /Users/cg/Library/Caches/go-build/17/17a37acb856cb4368bc6d6e5cf5f85de4b0e0a0d327c6503f2cc428067527bf9-d
/usr/local/go/pkg/tool/darwin_amd64/buildid -w $WORK/b001/exe/a.out # internal
mv $WORK/b001/exe/a.out hello
rm -r $WORK/b001/
```

```text
chugangdeMacBook-Pro:demo cg$ file /usr/local/go/pkg/darwin_amd64/runtime.a
/usr/local/go/pkg/darwin_amd64/runtime.a: current ar archive
chugangdeMacBook-Pro:demo cg$
```

/usr/local/go/src/cmd/link/internal/ld/main.go

### plan9 汇编

### [Go Assembly by Example](https://davidwong.fr/goasm/): Sqrt

[https://davidwong.fr/goasm/sqrt](https://davidwong.fr/goasm/sqrt)

### 自由职业

[https://chensuiyi.com/](https://chensuiyi.com/)

[有时间担心中年危机，还不如用忧虑的时间来提升自己——再论程序员该如何避免所谓的中年危机](https://www.cnblogs.com/JavaArchitect/p/11445017.html)

## macOS上的汇编入门（十二）——调试

[https://zhuanlan.zhihu.com/p/74627059](https://zhuanlan.zhihu.com/p/74627059)

## _**call指令\**_

_**当执行`call`指令时，进行两步操作：\**_ _**1）将当前的IP或CS和IP压入栈中\**_ _**2）转移\**_

_**`call`指令不能实现短转移，它的书写格式同`jmp`指令\**_

### _**依据标号进行转移的call指令\**_

_**语法格式：`call 标号`\**_ _**汇编解释：`(1) push IP (2) jmp near ptr 标号`\**_

### _**依据目的地址在指令中的call指令\**_

_**语法格式：`call far ptr 标号`\**_ _**汇编解释：`(1) push CS (2) push IP (3) jmp far ptr 标号`\**_

### _**转移地址在寄存器中的call指令\**_

_**语法格式：`call 16位reg`\**_ _**汇编解释：`(1) push IP (2) jmp 16位reg`\**_

### _**转移地址在内存中的call指令\**_

_**语法格式一：`call word ptr 内存单元地址`\**_ _**汇编解释一：`(1) push IP (2) jmp word ptr 内存单元地址`\**_

_**语法格式二：`call dword ptr 内存单元地址`\**_ _**汇编解释二：`(1) push CS (2) push IP (3) jmp dword ptr 内存单元地址`\**_

