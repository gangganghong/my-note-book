



# 用bison处理main函数

在action中写匹配规则很容易，难点是如何存储匹配到的语言元素。

另外一个难点是，怎么用C语言构造出合适的数据结构存储分析后的main函数。

例如，函数有多个参数，我用数组来存储这些参数。在分析前不知道参数个数，不能确定数组大小，但这个数组的大小却必须在定义AST数据结构时指定。

对这个问题，struct允许最后一个成员是不定长数组。那么，存储函数参数的问题勉强解决了。

可是，函数体由多个变量、多条语句组成，这些需要不定长数组来存储。一个结构体只允许最后一个成员是不定长数组。我如何才能存储函数体的变量和多条语句？

c语言作为一门写出过操作系统的语言，肯定能解决这类问题，可我却不知道怎么解决。



## 用链表

多个参数、变量、表达式这类场景，使用链表来存储。

前文的问题，消失了。

## 不明错误调试

执行报错，不知道错在哪里。耗费许久。耗费3个多小时无进展。

 excess elements in scalar initializer

```c
void *addToParamNodeList(struct ast *param, struct paramNode *paramNodeListHeader) {
    // todo 不能使用malloc初始化paramNode,只能如此声明。会出问题吗？
    struct paramNode *cur = {NULL, NULL};
    cur->param = param;
    cur->next = NULL;
    paramNodeCur->next = cur;
    paramNodeCur = cur;
    if (paramNodeListHeader->next == NULL) {
        paramNodeListHeader->next = paramNodeCur;
    }
}
```



```shell
Process 55480 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=1, address=0x0)
    frame #0: 0x0000000100003383 tcc`addToParamNodeList(param=0x0000000100304610, paramNodeListHeader=0x0000000000000000) at fb1-5funcs.c:192:16
   189  void *addToParamNodeList(struct ast *param, struct paramNode *paramNodeListHeader) {
   190      // todo 不能使用malloc初始化paramNode,只能如此声明。会出问题吗？
```



```shell
tcc was compiled with optimization - stepping may behave oddly; variables may not be available.
Process 65374 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = step over
    frame #0: 0x0000000100002959 tcc`yyparse at y.tab.c:1445:3 [opt]
Target 0: (tcc) stopped.
```

出现step over这条信息，不表示程序出错退出了调试。退出调试的标记是（遇到再补充）：

```shell
(lldb) n
tcc was compiled with optimization - stepping may behave oddly; variables may not be available.
Process 68664 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = step over
    frame #0: 0x0000000100002926 tcc`yyparse at y.tab.c:1455:3 [opt]
Target 0: (tcc) stopped.
```

断点调试退出的标志：`Process 68664 stopped` 。

```shell
(lldb) p *cur
(paramNode) $2 = {
  param = 0x0000000000000000
  next = 0x0000000000000000
}
```

内存地址 `0x0000000000000000` ，不理解。



```shell
fb1-5.y:90.6: warning: rule useless in parser due to conflicts [-Wother]
   90 | stmt:
      |      ^
```



```shell
 struct ast *createCon(struct ast *con) {
-> 13       struct ast *node = malloc(sizeof(struct ast));
   14       node->nodeType = con->nodeType;
   15       node->con = con;
   16       if (con->nodeType == 's' || con->nodeType == 'n') {
```



con的值是 's'，解析的字符串demo-A是：

```c
int main(int argc){int ab;if(true)ab=5;}
```

解析结果应该是 'true'。createStr的返回结果是正常的，不知道为何到了createCon，con的结果就发生了变化。

```shell
(lldb) p *con->nodeType
error: <user expression 23>:1:1: indirection requires pointer operand ('int' invalid)
```

This error occurs when you want to create pointer to your variable by `*my_var` instead of `&my_var`.

改成 ``(lldb) p con->nodeType``  就正确了。

### 小结

解析demo-A能一直执行到 ``createIfNode`` 。到了该函数的最后几步报错。另外，数字和字符串，都不能获取到正确的值。字符串获取到的值是's'，数字获取到的是一个比正确数值大得离谱的不知道怎么出来的值。createStr和createNum直到最后一步，返回的都是正确的结果。

接下来，该如何调试？不知道问题出现在哪个环节。

到目前为止，我一直在断点、看结果，距离关键出错点越来越来越近。

#### 简化问题

我还简化了问题，减少其他错误。

1. 目前规则不能匹配

```c
int main()
{
	if(true){
		ab = 5;
	}
}
```

我将输入数据简化为

````c
int main(int argc){int ab;if(true)ab=5;}
````

2. 不能匹配多个参数，我简化为只有一个参数、一个变量。

#### 障碍

1. createStr和createNum直到最后一步，返回的都是正确的结果。为啥到了其他地方，值变得不正确了？
2. ``createIfNode`` 。到了该函数的最后几步报错。

#### 其他

1. struct需要先定义，才能在诸如malloc(struct )等中使用。
2. 不能在函数外写这样的语句：

```c
struct ast *node = malloc(sizeof(struct ast));
```

### 调试的思路

已经耗费了3个小时。

让我重新开始，我会怎么做？

1. 编译，解决第一波错误，让代码能够编译成功。
2. 运行
   1. 语法错误，修改输入数据，简化它。
   2. 段错误，看错误信息中，有没有给出错误的代码位置。
      1. 有，直接使用lldb在保护该位置的函数设置断点。
      2. 无。弄清楚bison文件中函数的执行顺序，用二分法对函数设置断点，定位是哪个函数前后出了问题。

实际上，我是怎么做的？不太记得了，无章法，过程太琐碎了。似乎，我是挑了个最先执行的函数设置断点。

耗费了3个小时，又不能清楚地说出干了些啥。如何让别人、让以后的自己相信，这么长时间，我是在调试程序而不是在看视频？

无序的操作，很难给人留下印象。这是我通过调试这件事获得的认识。

找错误，断点跟踪，有可能会非常耗费时间，一定要想好思路，这是我获得的又一个认识。

现在，打算怎么做？

1. 断点时，发现struct不该有值的成员却有值，这是出错原因吗？
2. 审查一次代码。
   1. 收获很大，形参和实参数据类型不匹配。这是编译时的警告，之前我没留意看。解决了字符串和数字值不正确的问题。
   2. 实际上，只有这个导致了错误。
3. 再直接断点到createIfNode。
   1. 仍然到最后结果报错。
4. 弄清楚每个函数的执行顺序。
5. 没其他好方法。

Step Into和Step Over，Step Return有什么区别呢？

Step Into（F11/F5） 单步执行，遇到子函数就进入并且继续单步执行；

Step Over （F10/F6）在单步执行时，在函数内遇到子函数时不会进入子函数内单步执行，而是将子函数整个执行完在停止，也就是把子函数整个作为一步；

Step Return（Shift+F11/F7）在单步执行到子函数内时，用Step Return就可以执行完子函数余下部分，并返回上一层函数。

lldb 设置行号断点

breakpoint set -f <filename> -l <line number>

b set -f /Users/cg/data/code/study-compiler-java/study/flex-and-bison/flex/golang/fb1-5funcs.c -l 7

(lldb) b set -f /Users/cg/data/code/study-compiler-java/study/flex-and-bison/flex/golang/fb1-5funcs.c -l 7
Breakpoint 2: no locations (pending).
WARNING:  Unable to resolve breakpoint to any actual locations.

<!--这个语法不能使用-->

能指向到createIfNode，执行到return node出错退出。

是return node出错了，还是进入addToFuncStmtNodeList时出错？

在addToFuncStmtNodeList设置断点，看能否进入addToFuncStmtNodeList，若能，createIfNode无错。

不能进入addToFuncStmtNodeList，可是，createIfNode中的return node又能导致什么错误呢？

取消createIfNode断点，只设置addToFuncStmtNodeList，看结果。若能进入addToFuncStmtNodeList，那么，就很诡异了。

createIfNode、addToFuncStmtNodeList都没问题。

createIfNode-->addToFuncStmtNodeList-->createFunction-->newCode

三个函数~~可能~~是按上面的顺序执行，在它们上面设置断点。

```shell
  243  struct ast *createFunction(struct ast *returnType, struct ast *funcName,
   244                             struct paramNode *paramListHead, struct ast *funcBody) {
-> 245      struct ast *node = malloc(sizeof(struct ast));
   246      node->returnType = returnType->stringValue;
   247      node->funcName = funcName->stringValue;
   248      node->paramListHead = paramListHead;
Target 0: (tcc) stopped.
(lldb) p *returnType
(ast) $12 = {
  nodeType = 115
  l = 0x0000000000000000
  r = 0x000000006570000a
  tl = 0xaaaaaaaaaaaaaa00
  con = 0x0000000000000000
  el = 0x0000000000000000
  returnType = 0x20d6759d2bb10008 ""
  funcName = 0x0000000100000000 "����"
  paramListHead = 0x0000656c646e6168
  funcBody = 0x0000000100304140
  paramType = 0x0000000000000000
  paramName = 0x0000000000000000
  dataType = 0x20d6759d3b5f0005 ""
  variableName = 0x0000000100000000 "����"
  funcVariableListHead = 0x6574737973627573
  funcStmtsListHead = 0x000000000000006d
  numberValue = 0
  stringValue = 0x00000001003041c0 "int"
}
```



returnType中的funcName、variableName，有很奇怪的值。在哪里赋值的？它们不应该有值。

funcName很正常，

```shell
(lldb) p *funcName
(ast) $13 = {
  nodeType = 115
  l = 0x0000000000000000
  r = 0x00656e697475001a
  tl = 0x0000000000000000
  con = 0x0000000000000000
  el = 0x0000000000000000
  returnType = 0x0000000000000018 ""
  funcName = 0x0000000000000000
  paramListHead = 0x0000000100304510
  funcBody = 0x0000000000000068
  paramType = 0x0000000100304578 ""
  paramName = 0x0000000000000000
  dataType = 0x0000000000000000
  variableName = 0x0000000000000000
  funcVariableListHead = 0x0000000000000000
  funcStmtsListHead = 0x0000000000000000
  numberValue = 0
  stringValue = 0x00000001003043d0 "main"
}
```

```shell
# paramListHead->next->param 正常
(lldb) p *paramListHead->next->param
(ast) $16 = {
  nodeType = 112
  l = 0x0000000000000000
  r = 0x0000000000000000
  tl = 0x0000000000000000
  con = 0x0000000000000000
  el = 0x0000000000000000
  returnType = 0x0000000000000000
  funcName = 0x0000000000000000
  paramListHead = 0x0000000000000000
  funcBody = 0x0000000000000000
  paramType = 0x0000000100304480 "int"
  paramName = 0x0000000100304530 "argc"
  dataType = 0x0000000000000000
  variableName = 0x0000000000000000
  funcVariableListHead = 0x0000000000000000
  funcStmtsListHead = 0x0000000000000000
  numberValue = 0
  stringValue = 0x0000000000000000
}
```

```shell
# funcBody->funcVariableListHead->next->funcVariable 正常
(lldb) p *funcBody->funcVariableListHead->next->funcVariable
(ast) $20 = {
  nodeType = 118
  l = 0x0000000000000000
  r = 0x0000000000000000
  tl = 0x0000000000000000
  con = 0x0000000000000000
  el = 0x0000000000000000
  returnType = 0x0000000000000000
  funcName = 0x0000000000000000
  paramListHead = 0x0000000000000000
  funcBody = 0x0000000000000000
  paramType = 0x0000000000000000
  paramName = 0x0000000000000000
  dataType = 0x0000000100304560 "int"
  variableName = 0x0000000100304730 "ab"
  funcVariableListHead = 0x0000000000000000
  funcStmtsListHead = 0x0000000000000000
  numberValue = 0
  stringValue = 0x0000000000000000
}
```

```shell
(lldb) p *funcBody->funcStmtsListHead->next->funcStmtNode
(ast) $23 = {
  nodeType = 105
  l = 0x0000000000000000
  r = 0x00007fff9901007e
  tl = 0x0000000100304ba0
  con = 0x0000000100304950
  el = 0x0000000000000000
  # 不应该有值
  returnType = 0x00000001002054e0 "i"
  funcName = 0x0004000000000000 ""
  paramListHead = 0xffffffffffffffff
  funcBody = 0xffffffffffffffff
  paramType = 0x00000001002054f0 "~"
  paramName = 0x0000000000000000
  dataType = 0x4553555f46435f5f ""
  variableName = 0x455f545845545f52 ""
  # 不应该有值
  funcVariableListHead = 0x00474e49444f434e
  funcStmtsListHead = 0x0012000000000000
  numberValue = 1702057263
  stringValue = 0x762f6d766e2e2f67 ""
}
```

```shell
# funcBody->funcStmtsListHead->next->funcStmtNode->con->l 正常
(lldb) p *funcBody->funcStmtsListHead->next->funcStmtNode->con->l
(ast) $25 = {
  nodeType = 115
  l = 0x0000000000000000
  r = 0x0000000000000000
  tl = 0x0000000000000000
  con = 0x0000000000000000
  el = 0x0000000000000000
  returnType = 0x0000000000000000
  funcName = 0x0000000000000000
  paramListHead = 0x0000000000000000
  funcBody = 0x0000000000000000
  paramType = 0x0000000000000000
  paramName = 0x0000000000000000
  dataType = 0x0000000000000000
  variableName = 0x0000000000000000
  funcVariableListHead = 0x0000000000000000
  funcStmtsListHead = 0x0000000000000000
  numberValue = 0
  stringValue = 0x0000000100304890 "true"
}
```

```shell
# if 的 thenBody 正常
(lldb) p *funcBody->funcStmtsListHead->next->funcStmtNode->tl->l
(ast) $28 = {
  nodeType = 115
  l = 0x0000000000000000
  r = 0x0000000000000000
  tl = 0x0000000000000000
  con = 0x0000000000000000
  el = 0x0000000000000000
  returnType = 0x0000000000000000
  funcName = 0x0000000000000000
  paramListHead = 0x0000000000000000
  funcBody = 0x0000000000000000
  paramType = 0x0000000000000000
  paramName = 0x0000000000000000
  dataType = 0x0000000000000000
  variableName = 0x0000000000000000
  funcVariableListHead = 0x0000000000000000
  funcStmtsListHead = 0x0000000000000000
  numberValue = 0
  stringValue = 0x0000000100304940 "ab"
}

(lldb) p *funcBody->funcStmtsListHead->next->funcStmtNode->tl->r
(ast) $29 = {
  nodeType = 110
  l = 0x0000000000000000
  r = 0x0000000000000000
  tl = 0x0000000000000000
  con = 0x0000000000000000
  el = 0x0000000000000000
  returnType = 0x0000000000000000
  funcName = 0x0000000000000000
  paramListHead = 0x0000000000000000
  funcBody = 0x0000000000000000
  paramType = 0x0000000000000000
  paramName = 0x0000000000000000
  dataType = 0x0000000000000000
  variableName = 0x0000000000000000
  funcVariableListHead = 0x0000000000000000
  funcStmtsListHead = 0x0000000000000000
  numberValue = 5
  stringValue = 0x0000000000000000
}
```



结论：

```
createFunction
```

没有问题。

能下结论了，问题出在generateCCode。

验证方法，新增断点：在newCode设置断点，一路执行到newCode这个断点。能执行到newCode，说明在它前面执行的所有代码无问题，然后打印root结点看看，验证数据有没有问题。

经验证，newCode之前执行的代码，无问题。

重申结论：问题出在generateCCode。

所有构造AST的代码的bug，在修改createSte和createNum与使用它们的返回值数据类型匹配问题后，就已经全部解决了。

那么为什么耗费这么多时间调试到现在？

我不知道问题出在哪里。

误解了调试信息，就是下面这个调试信息：

```shell
tcc was compiled with optimization - stepping may behave oddly; variables may not be available.
Process 65374 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = step over
    frame #0: 0x0000000100002959 tcc`yyparse at y.tab.c:1445:3 [opt]
Target 0: (tcc) stopped.
```

在lldb中使用n执行下一步的时候，出现上面的信息，不能认定为当前代码的下一句出了问题。lldb的代码跳跃模式是`step over` ，执行n时，若遇到调用其他函数，不会进入被调用函数内部，而是把调用语句当作一句代码执行。

上面并不能很好解释，执行n，执行完了当前函数，一下子就执行到了newCode（最后面的代码）。

那么，我记住lldb这个工具具有下面的特性：

1. 使用n，执行完当前函数后，会跳过所有其他断点，直接执行所有代码。
2. 使用n，不会在下个断点停留。
3. 使用c，会在下个断点停留。

### 小结

用1小时28分，定位了出问题的地方。之前至少用了3个小时（绝对不止3个小时）。这次调试，是成功的。

经验：

1. 审查代码，认真看编译器给出的信息。
   1. 尤其是后者，代码复杂后，审查代码，心算能力、记忆能力不强，也许不能发现一些问题。可是结合编译器给出的信息，就能看出一些问题了。比如，这次，createStr和调用它的函数，形参和实参数据类型不一致，编译器已经给出了警告，我却视而不见。因为编译器给出了一大堆信息，都是警告。下次，即使是警告，也认真看一次，判断需不需要修改代码。多看一眼，我会节约三四个小时。
2. 对lldb给出信息的判断。
   1. lldb在哪个位置执行了n，程序就退出了，不代表，接下来那步就出了问题。n会一直执行完下面的所有代码（有机会再验证）。
   2. 我怎么知道它的n是这个意思？

## 功能

### 多个参数

#### char ef

如何匹配下面的代码？

```shell
int main(int argc,char ef,){int ab;if(true)ab=5;}
```

下面的规则，不行。

```basic
func:
	| identifier identifier '(' params ')' block	{ $$ = createFunction($1, $2, $4, $6); }
	;

//void *addToParamNodeList(struct ast *param, struct paramNode *paramNodeListHeader)
params:
	| params  param ','	{ addToParamNodeList($1, paramNodeListHeader); $$ = paramNodeListHeader;}
//	| param			{ addToParamNodeList($1, paramNodeListHeader); $$ = paramNodeListHeader;}
	;

//createParam(char *dataType, char *name)
param:
	| identifier identifier		{ $$ = createParam($1, $2); }
	;
```

已经尝试过的方案。

```
# A
params:
	| params  param ','	{ addToParamNodeList($1, paramNodeListHeader); $$ = paramNodeListHeader;}
//	| param			{ addToParamNodeList($1, paramNodeListHeader); $$ = paramNodeListHeader;}
	;
	
# B	
params:
  | params  param ','	{ addToParamNodeList($1, paramNodeListHeader); $$ = paramNodeListHeader;}
  //	| param			{ addToParamNodeList($1, paramNodeListHeader); $$ = paramNodeListHeader;}
  ;
  
 # C 
 params:
	| param  ','	params { addToParamNodeList($1, paramNodeListHeader); $$ = paramNodeListHeader;}
//	| param			{ addToParamNodeList($1, paramNodeListHeader); $$ = paramNodeListHeader;}
	;
```

打算怎么办？

1. 搜索？

   1. 非上策。这个问题比较依赖场景，搜索适合有固定场景和解决方案的问题，例如，编译器的报错信息，vue发送http请求等。

2. 看解析sql的demo D？

   1. 上面尝试过的方案中，我已经在规则中模仿D了，不成功。
   2. 也不好模仿，action不同。

3. 看bison书？

   1. 大海捞针。

4. 凭直觉换方案，然后一次次尝试，试图感动编译器？

   1. 这是我一贯的做法。绝对不行。
   2. 没想到方法。用直觉试了7分钟，不行。

5. 确定：下层的返回结果必须是构成上层的字符串吗，在循环中？比如：

   ```basic
   select_expr_list: select_expr { $$ = 1; }
       | select_expr_list ',' select_expr {$$ = $1 + 1; }
       | '*' { emit("SELECTALL"); $$ = 1; }
       ;
   
   select_expr: expr opt_as_alias ;
   ```

我的调试方法：

1. 先分析下面这样的简化版字符串

   ```basic
   int argc, char argv
   ```

   我更改了底层的返回结果，不行。初步认为，这是规则匹配与action的返回结果无关。

问题解决了，解决了问题，关键在于 /Users/cg/data/code/study-compiler-java/study/flex-and-bison/flex/golang/fb1-5.l 中没对逗号设置匹配规则，

```basic
"+" |
"-" |
"*" |
"/" |
"|" |
"(" |
")" |
"=" |
"{" |
"}" |
"," |
";"    {  return yytext[0]; }
```

之前搞错了导致bug的原因。

解决这个问题的关键方法是：先处理简化的问题，定位原因。

这个问题，大概耗费时间1个小时。虽然列出了几个思路，但我仍然犯了”凭直觉调试“的错误。

调试，有多浪费时间，又有多么的无重大价值，我很清楚。所以，再次提醒自己，先有思路，再调试，不允许凭直觉调试。

##### 小结

在bison中，有下面的规则：

1. $$ 必须与标识符F的数据类型一致，F的数据类型用%type定义。

2. action的结果不参与规则，不影响规则匹配数据。

3. action的结果，会参与上一层的action的运算。

   ```basic
   test:
   	| tparam ',' test		{ printf("test=%s\n", $1); $$ = contactStr($1, " ", $3); }
   	| tparam			{ $$ = $1; }
   	;
   tparam:
   	| identifier identifier	 { printf("tpamra=%s %s\n", $1->stringValue, $2->stringValue); $$ = contactStr($1->stringValue, " ", $2->stringValue);}
   	;
   ```

   tparam参与了test的action的运算，即，tparam这条规则的$$参与了test的action的运算。

##### 测试



1. 测试	`int main(int argc,char ef, double te){int ab;if(true)ab=5;}` ，使用`lldb` 命令：

   ```shell
   p *root->paramListHead->next->param
   p *root->paramListHead->next->next->param
   p *root->paramListHead->next->next->next->param
   ```

   结果符合预期。

#### 指针参数

todo

#### int &ab

todo

### 多变量

##### int ab

还算顺利。遇到一个因粗心导致的单链表创建错误问题。

处理逻辑和多参数非常相似，先对比了匹配规则和action，确定无误后，对比action使用的函数的具体内容。比较遗憾的是，没能很快看出单链表错误，而是对比正确代码后，才发现创建函数体变量的单链表代码有误。

1. 测试

   输入数据

   ````
   int main(int argc,char ef, double te){int ab;int mf;char hi;if(true)ab=5;}
   ````

   使用的lldb命令

   ```shell
   int main(int argc,char ef, double te){int ab;int mf;char hi;if(true)ab=5;}
   ```

耗费时间34分。

其他类型参数，再处理。

### 能解析包含换行符的代码

正常的代码包含缩进和换行，必须支持这个功能。

我用`##`来代替回车键作为输入测试代码时的结束标志。

```basic
		/*/Users/cg/data/code/study-compiler-java/study/flex-and-bison/flex/golang/fb1-5.l*/
"##"        { return EOL; }
[\r\n]		{  }
    /*       [\r\n] { return EOL; }      */

```



在flex中写注释，搜索资料，正确的方法是：

```
   # 行首必须有空白
   /*       [\r\n] { return EOL; }      */
```

耗费时间12分。以为行首必须有空白是在/*内部空白。这类问题，就应该直接搜索解决。

1. 测试

输入数据：

```c
int main(int argc, char ef, double te) {
    int ab;
    int mf;
    char hi;
    if (true)ab = 5;
}
```

```c
int main(int argc, char ef, double te) {
    int ab;
    int mf;
    char hi;
    if (true){ab = 5;}
}
```



使用的lldb命令

```shell
p  *root->funcBody->funcStmtsListHead->next->funcStmtNode->con->l
p  *root->funcBody->funcStmtsListHead->next->funcStmtNode->tl->l
p  *root->funcBody->funcStmtsListHead->next->funcStmtNode->tl->r   
p  *root->paramListHead->next->param  
p  *root->paramListHead->next->next->param 
p  *root->paramListHead->next->next->next->param     
p *root->funcBody->funcVariableListHead->next->funcVariable    
p  *root->funcBody->funcVariableListHead->next->next->funcVariable  
p  *root->funcBody->funcVariableListHead->next->next->next->funcVariable
```

正常。

耗费时间14分。

### 包含多条语句的if结构

thenBody 改成只有一个成员，每个成员是一个表达式，二元表达式。

elseBody也这样处理。

改规则，构思，多条语句，需要循环，好像总差点什么，不能很快写出来。

耗费时间25分。

写方案。

```basic
// /Users/cg/data/code/study-compiler-java/study/flex-and-bison/flex/golang/fb1-5.y
then:{}
	| '{' exprs '}'				{ $$ = $2; }
	| exprs					{ $$ = $1; }
	;

exprs:
	| expr ';' exprs			{ }
	;1
```

在括号等内部的部分，作为一个整体提取出来，处理起来更方便。

thenBody 改成只有一个成员，每个成员是一个表达式，二元表达式。

1. struct ast新增一个成员，exprNodeListHeader
2. 新增结构体，struct exprNode，两个成员，一个是struct ast *expr，另一个是struct exprNode *next。（边写边忘）。
3. 新增全局变量，struct exprNode *exprNodeListHeader，struct exprNode *exprNodeCur。

#### 测试

简化版

输入数据：

```c
if (true) {
  ab = 5;
  mf = 89;
  hi = 94;
}
```

使用规则stmt测试，直接看输出结果。

```basic
compilation_unit:{}
	| test	EOL			{ printf("test = %s\n", $1); }
//	| IF	EOL		   	{ printf("if = %s\n", $1);	}
//	| ELSE	EOL		   	{ printf("else = %s\n", $1);	}
//	| IF con EOL compilation_unit			{ dump($2); }
	| func EOL			{  newCode($1);				}
	| stmt				{  newCode($1); }
	
stmt:
	| then		{ $$ = $1; }
	| IF con then			{ $$ = createIfNode($2, $3, NULL); }
	| IF con then EOL		{ $$ = createIfNode($2, $3, NULL); }
	| IF con then ELSE else_body EOL		{ $$ = createIfNode($2, $3, $5); }
	;

con:							{}
	| expr						{ $$ = createCon( $1);}

//	| '(' IDENTIFIER '=' NUMBER	')'		{ $$ = createCon('+', $2, $4); }
//	| '(' IDENTIFIER '=' IDENTIFIER	')'		{ $$ = createCon('='); }
	;
//
then:{}
	| '{' exprs '}'				{ $$ = $2;  printf("exprs = %s\n", $2); }
	| exprs					{ $$ = $1; printf("exprs = %s\n", $1); }
	;

exprs:
	| expr ';' exprs			{ }
	;
```

需使用下面的打印函数才能看到结果

```c
// /Users/cg/data/code/study-compiler-java/study/flex-and-bison/flex/golang/fb1-5funcs.c
void printExpr(struct ast *expr){
    char nodeType = expr->nodeType;
    if(nodeType == 'e'){
        printExpr(expr->l);
        printExpr(expr->r);
    }else{
        // todo 先简化处理，固定表达式是 ab=5，右边是整型。
        printf("expr = %s %c %d\n", expr->l->stringValue, expr->nodeType, expr->r->numberValue);
    }
}1
```

输出

```c
chugangdeMacBook-Pro:golang cg$ ./tcc
if (true) {
        ab = 5;
        mf = 89;
        hi = 94;
    }
expr = hi = 94
expr = mf = 89
expr = ab = 5
exprs = =
```

证明匹配规则是正确的。

时间消耗50分。原因是，使用错误了验证方法，即错误的打印方法，该方法报错，时间花在了寻找错误原因上。没找到原因。后来，我直接放弃了这种验证方法，采用目前的验证方法。

不是必须解决的问题，可以跳过。解决了它，也没啥收益。

目前这个输出结果，已经可以了。后续可以在action中把这些表达式加入表达式单链表。

遇到问题时，调试，思路仍然不够清晰。如果一开始就选择第二种调试方法，那么，只需要10分钟，而不是50分钟！

有什么办法能够把测试代码分离出去，以便于维护和永久保存吗？

##### 验证elseBody规则

输入数据

```c
if (true) {
ab = 5;
mf = 89;
hi = 94;
}else{
_ab = 5;
mf_ = 89;
h_i = 94;
}##
```

基本通过了。测试输出结果让我很迷惑，第一次只能输出thenBody内的表达式，需要输出回车键后才能输出elseBody内的表达式。

耗费时间20分。原因，输出结果让我很疑惑。我因此多次重复执行。时间耗费在测试。

##### 验证

测试完整的if..else...结构。

完整版

输入数据

```c
int main(int argc, char ef, double te) {
    int ab;
    int mf;
    char hi;
    if (true) {
        ab = 5;
        mf = 89;
        hi = 94;
    }
}
```



耗费了很多时间才调试正确。约2个小时。

###### 小结

调试很不成功。一直强调，先有明确的思路，然后再运行代码。一遇到无头绪的问题，我就随意调试起来。运行代码，是件非常消耗时间的事情。

1. 断点调试的时候，我觉得，两次确定addToThenExprNodeList有没有问题，是能做到的。我试了很多次。

   ```c
   if (thenExprNodeListHeader->next == NULL) {
     	thenExprNodeListHeader->next = thenExprNodeCur;
   }
   ```

   一次断点运行，就应该看出是这里出了问题。

2. 修改好addToThenExprNodeList后，thenExprNodeListHeader->next  无问题。
3. 查看 `p  *root->funcBody->funcStmtsListHead->next->funcStmtNode` ，con值正常，说明，createIfNode无问题。
4. 查看 `p  *root->funcBody->funcStmtsListHead->next->funcStmtNode->elseExprNodeListHeader->next` ，异常。说明问题出在 createIfNode 中对`elseExprNodeListHeader` 和 `thenExprNodeListHeader` 的处理上。
5. 最后，发现，问题出在 第4步 ，以及 `then` 和 `else_body` 的action上。

时间消耗在，断点调试多次，却未能定位问题。

### 杂项

block不支持不带花括号的代码。暂未想到实现不带花括号的block的规则。







