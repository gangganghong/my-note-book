# 第2章 词法分析

终结符号：`if`、`(`这类词法元素。

非终结符号：`expr`和`stmt`这类。

产生式：`stmt -> if(expr) stmt else stmt`中`->`右边部分就是产生式。



上下文无关文法有4个组成要素：

1. 终结符号集合
2. 非终结符号集合
3. 产生式集合
   1. 产生式头部或左部
   2. 箭头
   3. 产生式主体或右部
4. 指定一个非终结符号为开始符号

`stmt -> if expr stmt | 2`，这是一个产生式P。P是非终结符号`stmt`的产生式。

`expr -> 2 + 3`，这也是一个产生式，记作A。把P中的`expr`替换为A，这个过程叫推导。

`stmt -> if 2 + 3 stmt`。

`stmt -> if 2 + 3 2`，记作L。L是产生式P的语言。

语法分析的任务是：给出一个终结符号，找出从开始符号推导出这个终结符号的方法。如果不能从开始符号推导出这个终结符号，就认为发生了语法错误。

`+、-、*、/`是左结合的，指数运算、`=`是右结合的。

左结合是指先计算左边的。`9+8-2`，先计算`9+8`，再计算`-2`，等价于`(9+8)-2`。

右结合是指先计算右边的。`a=b=c`，计算过程：

1. 先计算`b=c`。
2. 把`b=c`的结果赋值给`a`。
3. 等价于`a = (b=c)`。

优先级在文法中的体现是，优先级高的在上一个产生式中用非终结符表示，在下一个产生中替换为另一个非终结符或终结符。

结合性如何在文法中体现，我还没有弄清楚。

```shell
expr -->	expr + term | expr - term | term
term -->	term * factor | term / factor | factor
factor --> digit | expr
```

`factor`是因子。任何用括号括起来的组成部分都是因子。

通过在第一个产生式中使用`term`，第二个产生式是`term`的产生式，实现优先级。

结合性，我仍然没有找到。

## 语法制导定义

一个文法的产生式（比如中缀表达式），只能反映现有表达式的组成，就像正则表达式一样。

结合语法制导定义，能把同一个文法的产生式翻译成另外一种形式。例如，能把中缀表达式翻译成后缀表达式。

同一个文法的产生式，结合不同的语法制导定义，能翻译成不同的形式。

一个`1`，加上单位，可能是`1厘米`、`1分钟`、`1元钱`。语法制导定义的类似`单位`。

## 翻译方案

![image-20210315140139902](/Users/cg/Documents/gitbook/my-note-book/bian-yi-qi-she-ji/note/image-20210315140139902.png)

如何看图得到中缀表达式？只看实线。用代码怎么实现呢？

后缀表达式呢？只看虚线。

## match

```c
void match(int t){
  	// lookahead 是全局变量
  	// 很有迷惑性。形参是t，实参是lookahead。但t与全局变量lookahead并非总是相同。
  	if(lookahead == t){
      	// 读取下一个字符
      	getchar();
    }else{
      	// 出错
    }
}
```

## 疑问

用中缀表达式为例子。

一个中缀表达式，用抽象语法树表示出来，记作A。

和这个中缀表达式等价的后缀表达式，用抽象语法树表示出来，记作B。

用代码实现B。

A变为B，是怎么做到的？目前，我是人工计算出结果，直接写出B。

A是输入代码的抽象语法树，是需要解析的对象。

A变成B，肯定不能通过人工计算，而是需要用代码实现。理由很简单。要解析一段代码，不能先人工把它转换成等价的另一种形式，然后再用代码来解析另一种形式。

后缀表达式，不是最终形式，最终结果是一颗抽象语法树。

A怎么变成B？

## 符号表

```c
int a;int b;bool c;
```

符号表：

```c
{a:int},{b:int},{c:boo}
```

符号表，就是一个哈希表，这个哈希表中的元素是键值对，键是变量名，值是变量的数据类型。

符号表会反映变量的作用域。怎么反映的？

每一块程序都有一个对应的符号表。

符号表是数据结构，这个数据保存源代码的各种信息。

从符号表，能获得源代码的各种信息，甚至，能根据符号表还原源代码。

符号存储在栈中或散列表中。

嵌套块，栈的当前元素是当前块的符号表。栈的下一个元素，存储当前块的父块。

不理解怎么使用这个存储符号表的栈。

![image-20210316164402709](/Users/cg/Documents/gitbook/my-note-book/bian-yi-qi-she-ji/note/image-20210316164402709.png)

怎么看这个图？这个图画得非常不好，不看注释，很难正确理解。

第二列折行，是因为写不下，正常的应该是下面这样的，只举一个例子：

```c
block		--->		{ saved = top; top = new Env(top); print("{}");	}	
								'{'decls stmts'}'		
                { top = saved; print("} "); }
```

怎么实现作用域呢？

1. 在进入程序块之前，把`top`存储在`saved`中。`top`是上一个程序块中的符号表。
2. 创建一个新符号表，作为当前程序块的符号表，并存储在`top`中。在当前程序块中，都可以从`top`中获取符号表。
3. 离开当前程序块之前，从`saved`中获取上一个程序块的符号表，并且存储到`top`中。

## 中间代码

抽象语法树（可能不准确）和三目运算符，都是中间代码。

从前面的翻译表达式生成抽象语法树，就在这个小节。

概念很混乱。

翻译表达式是什么？

抽象语法树又是什么？

不清楚这些概念也不是大问题，看到一个表达式，能指出是抽象表达式还是翻译表达式，就足够了。

然而，不能说出教科书定义或用自己的语言说出定义，就无法向别人证明自己懂这个东西。

## 语法分析

![image-20210317111146995](/Users/cg/Documents/gitbook/my-note-book/bian-yi-qi-she-ji/note/image-20210317111146995.png)





![image-20210317111219678](/Users/cg/Documents/gitbook/my-note-book/bian-yi-qi-she-ji/note/image-20210317111219678.png)![image-20210317111219771](/Users/cg/Documents/gitbook/my-note-book/bian-yi-qi-she-ji/note/image-20210317111219771.png)



![image-20210317175527930](/Users/cg/Documents/gitbook/my-note-book/bian-yi-qi-she-ji/note/image-20210317175527930.png)





![image-20210317175605619](/Users/cg/Documents/gitbook/my-note-book/bian-yi-qi-she-ji/note/image-20210317175605619.png)



![image-20210317183338523](/Users/cg/Documents/gitbook/my-note-book/bian-yi-qi-she-ji/note/image-20210317183338523.png)	

## golang 编译过程

file /usr/lib/golang/pkg/tool/linux_amd64/link
/usr/lib/golang/pkg/tool/linux_amd64/link: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, not stripped

```shell
[root@localhost code]# go run -x println.go > println-run.log
WORK=/tmp/go-build215017891
mkdir -p $WORK/b001/
cat >$WORK/b001/importcfg << 'EOF' # internal
# import config
packagefile runtime=/usr/lib/golang/pkg/linux_amd64/runtime.a
EOF
cd /home/cg/code
/usr/lib/golang/pkg/tool/linux_amd64/compile -o $WORK/b001/_pkg_.a -trimpath "$WORK/b001=>" -p main -complete -buildid qyAdWfxP1x7okpm7mIlT/qyAdWfxP1x7okpm7mIlT -dwarf=false -goversion go1.14.7 -D _/home/cg/code -importcfg $WORK/b001/importcfg -pack -c=2 ./println.go
/usr/lib/golang/pkg/tool/linux_amd64/buildid -w $WORK/b001/_pkg_.a # internal
cp $WORK/b001/_pkg_.a /root/.cache/go-build/a6/a67c60514779d94d011a93b68e26c5437235f1a264e67db58fe54fdf265c9db4-d # internal
cat >$WORK/b001/importcfg.link << 'EOF' # internal
packagefile command-line-arguments=$WORK/b001/_pkg_.a
packagefile runtime=/usr/lib/golang/pkg/linux_amd64/runtime.a
packagefile internal/bytealg=/usr/lib/golang/pkg/linux_amd64/internal/bytealg.a
packagefile internal/cpu=/usr/lib/golang/pkg/linux_amd64/internal/cpu.a
packagefile runtime/internal/atomic=/usr/lib/golang/pkg/linux_amd64/runtime/internal/atomic.a
packagefile runtime/internal/math=/usr/lib/golang/pkg/linux_amd64/runtime/internal/math.a
packagefile runtime/internal/sys=/usr/lib/golang/pkg/linux_amd64/runtime/internal/sys.a
EOF
mkdir -p $WORK/b001/exe/
cd .
/usr/lib/golang/pkg/tool/linux_amd64/link -o $WORK/b001/exe/println -importcfg $WORK/b001/importcfg.link -s -w -buildmode=exe -buildid=u-qCTeFd7HuAjNI4BIuJ/qyAdWfxP1x7okpm7mIlT/9zPagCUSSVC6tgbcSgN3/u-qCTeFd7HuAjNI4BIuJ -extld=gcc $WORK/b001/_pkg_.a
$WORK/b001/exe/println
helloworld
```



```shell
[root@localhost code]# file /usr/lib/golang/pkg/tool/linux_amd64/compile
/usr/lib/golang/pkg/tool/linux_amd64/compile: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, not stripped
```



现代编译器原理一般不包括实现汇编器。我要写的golang编译器，只需完成词法分析、语法分析、生成汇编代码、编译、链接。

```shell
[root@localhost code]# go tool compile -o main.o println.go
[root@localhost code]# file main.o
main.o: current ar archive
[root@localhost code]# go tool link -o main -buildmode=exe main.o
[root@localhost code]# file main
main: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, not stripped
```



不写汇编器。据说，无技术含量。而且，投入产出比不高。一定不能写汇编器。

相关资料：

如何编写将汇编代码翻译成机器码的程序？

https://www.zhihu.com/question/26659135

《Go语言设计与实现》

https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-machinecode/#%E6%B1%87%E7%BC%96%E5%99%A8

Golang编译器漫谈（2）编译器目标文件

https://hao.io/2020/01/golang%e7%bc%96%e8%af%91%e5%99%a8%e6%bc%ab%e8%b0%88%ef%bc%882%ef%bc%89%e7%bc%96%e8%af%91%e5%99%a8%e7%9b%ae%e6%a0%87%e6%96%87%e4%bb%b6/

手把手教你做一个 C 语言编译器

https://wizardforcel.gitbooks.io/diy-c-compiler/content/5.html

## 名校公开课

OCW---麻省理工

https://ocw.mit.edu/courses/translated-courses/traditional-chinese/

X86实验

https://pdos.csail.mit.edu/6.828/2017/schedule.html

## 学习方法读书笔记

两本书：A（象棋冠军）、B（教授）。

A大量使用象棋细节，我不理解；排版紧凑；行文环环相扣，一边读一边理解不了。

B使用日常生活小事，排版宽松。

不要指望一口气学成一个胖子。学得越好就越喜欢学习。

人天生具备非凡的心算能力，例如打羽毛球时接球。接羽毛球还能验证这个，奇特。

这本书对学习数学有用。

先快速浏览全文，然后再逐字逐句读。

专注思维，在一段时间内，顺着有条理的思路，聚焦一个问题。发散思维，不刻意，不沿着任何思路。发散思维，在休息和零碎时间进行。发散思维不能凭空产生，在专注思维打下的基础上。

发散思维，像手电筒发出的散光；专注思维，像激光灯发出的聚焦的“光针”。

学新事物，先用发散思维，方法论是：先广泛、粗略地了解，再选择一块专注了解。

数学等理科为啥难学？

1. 抽象。多训练。
2. 思维定势。

写吉他曲子，越是专注写出来的曲子越味同嚼蜡。不赞同。我文章，都是刻意写出来的。文章不会自然而然产生。

集中思考一个问题没有进展，放松一下有助于解决问题。赞同这点。

学习是在困惑中寻找答案的过程。当你能描述出困惑你的问题是什么时，你就成功了一半。只要你发现了困惑你的东西是什么，那么你离解答出来就不远了。



## 小插曲

用`flow`画流程图。

https://www.jianshu.com/p/9f60696995e5

编译原理中的文法用什么语言会高亮？

```flow
st=>start: Start
e=>end
op=>operation: My Operation
cond=>condition: Yes or No?

st->op->cond
cond(yes)->e
cond(no)->op
```

```flow
start1=>start: 初始设计
op1=>operation: P=0
op2=>operation: 结构分析与敏度分析
op3=>operation: 建立原问题（Primal Problem）
op4=>operation: 建立近似问题（Approximate Proble）
op5=>operation: 求解近似问题，得到$X^{P+1}$
cond1=>condition: 是否小于允许误差
op6=>operation: P=P+1
end=>end: 结束

start1->op1->op2->op3->op4->op5->cond1
cond1(no)->op6(top)->op2
cond1(yes)->end
```



