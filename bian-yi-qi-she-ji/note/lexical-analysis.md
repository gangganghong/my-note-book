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



