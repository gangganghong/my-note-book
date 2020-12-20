# 用bison处理main函数

## 用bison处理main函数

### 进度快照

1. 项目代码在：

/Users/cg/data/code/study-compiler-java/study/flex-and-bison/flex/golang

在for-todo分支，代码仓库：[https://github.com/gangganghong/study-compiler-java](https://github.com/gangganghong/study-compiler-java)

1. 修改完项目后，使用 make 编译，使用 make clean 清理目前的编译结果，使用make run 运行设定的程序。
   1. 目前，我没有使用make run
2. 使用lldb断点调试程序。
   1. 使用make编译代码，得到可执行文件 tcc。
   2. 使用lldb断点调试程序，例如：lldb tcc。
   3. 设置断点：b newCode
   4. 让程序运行：run
   5. 输入数据：在run执行后，在命令行窗口粘贴数据。
   6. 打印变量：p \*root
   7. 执行下一步：n
   8. 执行到下个断点：c
3. 目前，我在newCode设置断点，使用 p 打印 root 的成员来观察AST是否正确。
4. 正在做解析for结构的功能，未完成，中止了。

在action中写匹配规则很容易，难点是如何存储匹配到的语言元素。

另外一个难点是，怎么用C语言构造出合适的数据结构存储分析后的main函数。

例如，函数有多个参数，我用数组来存储这些参数。在分析前不知道参数个数，不能确定数组大小，但这个数组的大小却必须在定义AST数据结构时指定。

对这个问题，struct允许最后一个成员是不定长数组。那么，存储函数参数的问题勉强解决了。

可是，函数体由多个变量、多条语句组成，这些需要不定长数组来存储。一个结构体只允许最后一个成员是不定长数组。我如何才能存储函数体的变量和多条语句？

c语言作为一门写出过操作系统的语言，肯定能解决这类问题，可我却不知道怎么解决。

### 用链表

多个参数、变量、表达式这类场景，使用链表来存储。

前文的问题，消失了。

### 不明错误调试

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

```text
Process 55480 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=1, address=0x0)
    frame #0: 0x0000000100003383 tcc`addToParamNodeList(param=0x0000000100304610, paramNodeListHeader=0x0000000000000000) at fb1-5funcs.c:192:16
   189  void *addToParamNodeList(struct ast *param, struct paramNode *paramNodeListHeader) {
   190      // todo 不能使用malloc初始化paramNode,只能如此声明。会出问题吗？
```

```text
tcc was compiled with optimization - stepping may behave oddly; variables may not be available.
Process 65374 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = step over
    frame #0: 0x0000000100002959 tcc`yyparse at y.tab.c:1445:3 [opt]
Target 0: (tcc) stopped.
```

出现step over这条信息，不表示程序出错退出了调试。退出调试的标记是（遇到再补充）：

```text
(lldb) n
tcc was compiled with optimization - stepping may behave oddly; variables may not be available.
Process 68664 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = step over
    frame #0: 0x0000000100002926 tcc`yyparse at y.tab.c:1455:3 [opt]
Target 0: (tcc) stopped.
```

断点调试退出的标志：`Process 68664 stopped` 。

```text
(lldb) p *cur
(paramNode) $2 = {
  param = 0x0000000000000000
  next = 0x0000000000000000
}
```

内存地址 `0x0000000000000000` ，不理解。

```text
fb1-5.y:90.6: warning: rule useless in parser due to conflicts [-Wother]
   90 | stmt:
      |      ^
```

```text
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

```text
(lldb) p *con->nodeType
error: <user expression 23>:1:1: indirection requires pointer operand ('int' invalid)
```

This error occurs when you want to create pointer to your variable by `*my_var` instead of `&my_var`.

改成 `(lldb) p con->nodeType` 就正确了。

#### 小结

解析demo-A能一直执行到 `createIfNode` 。到了该函数的最后几步报错。另外，数字和字符串，都不能获取到正确的值。字符串获取到的值是's'，数字获取到的是一个比正确数值大得离谱的不知道怎么出来的值。createStr和createNum直到最后一步，返回的都是正确的结果。

接下来，该如何调试？不知道问题出现在哪个环节。

到目前为止，我一直在断点、看结果，距离关键出错点越来越来越近。

**简化问题**

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

```c
int main(int argc){int ab;if(true)ab=5;}
```

1. 不能匹配多个参数，我简化为只有一个参数、一个变量。

**障碍**

1. createStr和createNum直到最后一步，返回的都是正确的结果。为啥到了其他地方，值变得不正确了？
2. `createIfNode` 。到了该函数的最后几步报错。

**其他**

1. struct需要先定义，才能在诸如malloc\(struct \)等中使用。
2. 不能在函数外写这样的语句：

```c
struct ast *node = malloc(sizeof(struct ast));
```

#### 调试的思路

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

breakpoint set -f -l

b set -f /Users/cg/data/code/study-compiler-java/study/flex-and-bison/flex/golang/fb1-5funcs.c -l 7

\(lldb\) b set -f /Users/cg/data/code/study-compiler-java/study/flex-and-bison/flex/golang/fb1-5funcs.c -l 7 Breakpoint 2: no locations \(pending\). WARNING: Unable to resolve breakpoint to any actual locations.

能指向到createIfNode，执行到return node出错退出。

是return node出错了，还是进入addToFuncStmtNodeList时出错？

在addToFuncStmtNodeList设置断点，看能否进入addToFuncStmtNodeList，若能，createIfNode无错。

不能进入addToFuncStmtNodeList，可是，createIfNode中的return node又能导致什么错误呢？

取消createIfNode断点，只设置addToFuncStmtNodeList，看结果。若能进入addToFuncStmtNodeList，那么，就很诡异了。

createIfNode、addToFuncStmtNodeList都没问题。

createIfNode--&gt;addToFuncStmtNodeList--&gt;createFunction--&gt;newCode

三个函数~~可能~~是按上面的顺序执行，在它们上面设置断点。

```text
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

```text
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

```text
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

```text
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

```text
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

```text
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

```text
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

```text
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

```text
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

#### 小结

用1小时28分，定位了出问题的地方。之前至少用了3个小时（绝对不止3个小时）。这次调试，是成功的。

经验：

1. 审查代码，认真看编译器给出的信息。
   1. 尤其是后者，代码复杂后，审查代码，心算能力、记忆能力不强，也许不能发现一些问题。可是结合编译器给出的信息，就能看出一些问题了。比如，这次，createStr和调用它的函数，形参和实参数据类型不一致，编译器已经给出了警告，我却视而不见。因为编译器给出了一大堆信息，都是警告。下次，即使是警告，也认真看一次，判断需不需要修改代码。多看一眼，我会节约三四个小时。
2. 对lldb给出信息的判断。
   1. lldb在哪个位置执行了n，程序就退出了，不代表，接下来那步就出了问题。n会一直执行完下面的所有代码（有机会再验证）。
   2. 我怎么知道它的n是这个意思？

### 功能

#### 多个参数

**char ef**

如何匹配下面的代码？

```text
int main(int argc,char ef,){int ab;if(true)ab=5;}
```

下面的规则，不行。

```text
func:
    | identifier identifier '(' params ')' block    { $$ = createFunction($1, $2, $4, $6); }
    ;

//void *addToParamNodeList(struct ast *param, struct paramNode *paramNodeListHeader)
params:
    | params  param ','    { addToParamNodeList($1, paramNodeListHeader); $$ = paramNodeListHeader;}
//    | param            { addToParamNodeList($1, paramNodeListHeader); $$ = paramNodeListHeader;}
    ;

//createParam(char *dataType, char *name)
param:
    | identifier identifier        { $$ = createParam($1, $2); }
    ;
```

已经尝试过的方案。

```text
# A
params:
    | params  param ','    { addToParamNodeList($1, paramNodeListHeader); $$ = paramNodeListHeader;}
//    | param            { addToParamNodeList($1, paramNodeListHeader); $$ = paramNodeListHeader;}
    ;

# B    
params:
  | params  param ','    { addToParamNodeList($1, paramNodeListHeader); $$ = paramNodeListHeader;}
  //    | param            { addToParamNodeList($1, paramNodeListHeader); $$ = paramNodeListHeader;}
  ;

 # C 
 params:
    | param  ','    params { addToParamNodeList($1, paramNodeListHeader); $$ = paramNodeListHeader;}
//    | param            { addToParamNodeList($1, paramNodeListHeader); $$ = paramNodeListHeader;}
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

   ```text
   select_expr_list: select_expr { $$ = 1; }
       | select_expr_list ',' select_expr {$$ = $1 + 1; }
       | '*' { emit("SELECTALL"); $$ = 1; }
       ;

   select_expr: expr opt_as_alias ;
   ```

我的调试方法：

1. 先分析下面这样的简化版字符串

   ```text
   int argc, char argv
   ```

   我更改了底层的返回结果，不行。初步认为，这是规则匹配与action的返回结果无关。

问题解决了，解决了问题，关键在于 /Users/cg/data/code/study-compiler-java/study/flex-and-bison/flex/golang/fb1-5.l 中没对逗号设置匹配规则，

```text
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

**小结**

在bison中，有下面的规则：

1. $$ 必须与标识符F的数据类型一致，F的数据类型用%type定义。
2. action的结果不参与规则，不影响规则匹配数据。
3. action的结果，会参与上一层的action的运算。

   ```text
   test:
       | tparam ',' test        { printf("test=%s\n", $1); $$ = contactStr($1, " ", $3); }
       | tparam            { $$ = $1; }
       ;
   tparam:
       | identifier identifier     { printf("tpamra=%s %s\n", $1->stringValue, $2->stringValue); $$ = contactStr($1->stringValue, " ", $2->stringValue);}
       ;
   ```

   tparam参与了test的action的运算，即，tparam这条规则的$$参与了test的action的运算。

**测试**

1. 测试 `int main(int argc,char ef, double te){int ab;if(true)ab=5;}` ，使用`lldb` 命令：

   ```text
   p *root->paramListHead->next->param
   p *root->paramListHead->next->next->param
   p *root->paramListHead->next->next->next->param
   ```

   结果符合预期。

**指针参数**

todo

**int &ab**

todo

#### 多变量

**int ab**

还算顺利。遇到一个因粗心导致的单链表创建错误问题。

处理逻辑和多参数非常相似，先对比了匹配规则和action，确定无误后，对比action使用的函数的具体内容。比较遗憾的是，没能很快看出单链表错误，而是对比正确代码后，才发现创建函数体变量的单链表代码有误。

1. 测试

   输入数据

   ```text
   int main(int argc,char ef, double te){int ab;int mf;char hi;if(true)ab=5;}
   ```

   使用的lldb命令

   ```text
   int main(int argc,char ef, double te){int ab;int mf;char hi;if(true)ab=5;}
   ```

耗费时间34分。

其他类型参数，再处理。

#### 能解析包含换行符的代码

正常的代码包含缩进和换行，必须支持这个功能。

我用`##`来代替回车键作为输入测试代码时的结束标志。

```text
        /*/Users/cg/data/code/study-compiler-java/study/flex-and-bison/flex/golang/fb1-5.l*/
"##"        { return EOL; }
[\r\n]        {  }
    /*       [\r\n] { return EOL; }      */
```

在flex中写注释，搜索资料，正确的方法是：

```text
   # 行首必须有空白
   /*       [\r\n] { return EOL; }      */
```

耗费时间12分。以为行首必须有空白是在/\*内部空白。这类问题，就应该直接搜索解决。

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

```text
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

#### 包含多条语句的if结构

thenBody 改成只有一个成员，每个成员是一个表达式，二元表达式。

elseBody也这样处理。

改规则，构思，多条语句，需要循环，好像总差点什么，不能很快写出来。

耗费时间25分。

写方案。

```text
// /Users/cg/data/code/study-compiler-java/study/flex-and-bison/flex/golang/fb1-5.y
then:{}
    | '{' exprs '}'                { $$ = $2; }
    | exprs                    { $$ = $1; }
    ;

exprs:
    | expr ';' exprs            { }
    ;1
```

在括号等内部的部分，作为一个整体提取出来，处理起来更方便。

thenBody 改成只有一个成员，每个成员是一个表达式，二元表达式。

1. struct ast新增一个成员，exprNodeListHeader
2. 新增结构体，struct exprNode，两个成员，一个是struct ast _expr，另一个是struct exprNode_ next。（边写边忘）。
3. 新增全局变量，struct exprNode _exprNodeListHeader，struct exprNode_ exprNodeCur。

**测试**

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

```text
compilation_unit:{}
    | test    EOL            { printf("test = %s\n", $1); }
//    | IF    EOL               { printf("if = %s\n", $1);    }
//    | ELSE    EOL               { printf("else = %s\n", $1);    }
//    | IF con EOL compilation_unit            { dump($2); }
    | func EOL            {  newCode($1);                }
    | stmt                {  newCode($1); }

stmt:
    | then        { $$ = $1; }
    | IF con then            { $$ = createIfNode($2, $3, NULL); }
    | IF con then EOL        { $$ = createIfNode($2, $3, NULL); }
    | IF con then ELSE else_body EOL        { $$ = createIfNode($2, $3, $5); }
    ;

con:                            {}
    | expr                        { $$ = createCon( $1);}

//    | '(' IDENTIFIER '=' NUMBER    ')'        { $$ = createCon('+', $2, $4); }
//    | '(' IDENTIFIER '=' IDENTIFIER    ')'        { $$ = createCon('='); }
    ;
//
then:{}
    | '{' exprs '}'                { $$ = $2;  printf("exprs = %s\n", $2); }
    | exprs                    { $$ = $1; printf("exprs = %s\n", $1); }
    ;

exprs:
    | expr ';' exprs            { }
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

**验证elseBody规则**

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

**验证**

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

**小结**

调试很不成功。一直强调，先有明确的思路，然后再运行代码。一遇到无头绪的问题，我就随意调试起来。运行代码，是件非常消耗时间的事情。

1. 断点调试的时候，我觉得，两次确定addToThenExprNodeList有没有问题，是能做到的。我试了很多次。

   ```c
   if (thenExprNodeListHeader->next == NULL) {
         thenExprNodeListHeader->next = thenExprNodeCur;
   }
   ```

   一次断点运行，就应该看出是这里出了问题。

2. 修改好addToThenExprNodeList后，thenExprNodeListHeader-&gt;next 无问题。
3. 查看 `p  *root->funcBody->funcStmtsListHead->next->funcStmtNode` ，con值正常，说明，createIfNode无问题。
4. 查看 `p  *root->funcBody->funcStmtsListHead->next->funcStmtNode->elseExprNodeListHeader->next` ，异常。说明问题出在 createIfNode 中对`elseExprNodeListHeader` 和 `thenExprNodeListHeader` 的处理上。
5. 最后，发现，问题出在 第4步 ，以及 `then` 和 `else_body` 的action上。

时间消耗在，断点调试多次，却未能定位问题。

#### 测试多个if语句

输入：

```c
int main(int argc, char ef, double te) {
    int ab;
    int mf;
    char hi;
    if (true) {
        ab = 5;
        test_mf = 89;
        hi = 94;
    }else{
        _ab = 5;
        mf_ = 89;
        h_i = 94;
    }
    if (false) {
        ab_ = 5;
        mf_ = 89;
        hi_ = 94;
    }else{
        _ab__ = 5;
        mf___ = 89;
        h_i__ = 94;
    }
}##
```

p \*funcStmtNodeListHeader-&gt;next-&gt;next-&gt;funcStmtNode-&gt;thenExprNodeListHeader-&gt;next-&gt;next-&gt;expr-&gt;l

输出的总是第一个if结构的数据，原因是，存储if结构的变量头结点的变量是全局变量，所以if结构的变量头结点使用的都是这个全局变量。

耗费时间58分，才定位出问题原因。

原因：一是分心，二是没有及时断点到addToFuncStmtNodeList，三是观察断点数据时没查看全部。

多个if结构的con正常，只是结构内的变量异常。

怎么解决？

不使用全局变量存储单链表的头结点，在`addToParamNodeList` 等函数内部使用`static`变量存储单链表的头结点。全局变量默认是`static`。`static` 变量不会每次都初始化，只初始化一次。

修改起来，也必将麻烦，会按下了葫芦飘起来了瓢。

目前的代码，全写在一个文件里，我感觉有点杂乱了。还有那么多语言结构（已知的、未知的）需要解析，但我又懒得列出一个功能表。

C语言不支持这种语法

```c
static struct paramNode *paramNode = (struct paramNode *) malloc(sizeof(struct paramNode));
```

我不知道该怎么办了。

```text
 indirection requires pointer operand ('paramNode' invalid)
```

```text
 # 能用的表达式
 p *root->paramListHead.next->next->param
 p *root->funcBody->funcStmtsListHead.next->next->funcStmtNode->con->l
 p *root->funcBody->funcStmtsListHead.next->next->funcStmtNode->thenExprNodeListHeader.next->expr->r
```

无法运行`p *root->funcBody->funcStmtsListHead.` ，会报错：

```text
(lldb) p *root->funcBody->funcStmtsListHead
error: <user expression 62>:1:1: indirection requires pointer operand ('funcStmtNode' invalid)
*root->funcBody->funcStmtsListHead
^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
(lldb)
```

改成`p *root->funcBody->funcStmtsListHead.next` 就可以。原因不明。

用这种方式，能正确存储AST，然而，这种不规则的读数据方法，我觉得难维护。

其实也不是特别没有规则，读取指针成员用`->`，读取非指针成员用`.`。让我觉得不规则的地方是，其他变量能用p直接打印，唯独非指针不能用p打印。

使用非指针，是为了解决全局变量在函数内部重新初始化但又不能重设已经获取的值的问题。一部分代码如下：

```c
struct ast *createFunction(struct ast *returnType, struct ast *funcName,
                           struct paramNode *paramListHead, struct ast *funcBody) {
    struct ast *node = malloc(sizeof(struct ast));
    node->returnType = returnType->stringValue;
    node->funcName = funcName->stringValue;
    node->paramListHead = *paramListHead;
    node->funcBody = funcBody;

    paramListHead->next = NULL;

    return node;
}
```

指针在一块内存存储的是一个内存地址A，根据A，能读取到A地址存储的数据。

指针，有三个数据，第一个，存储这个指针P的内存地址A（任何变量或数据，都需要放入一块内存中）；第二个，内存A存储的数据B，B是指针的值，是一个内存地址；第三个，存放在内存B处的数据C。

将指针A存储的值（是一个内存地址），赋值给另一个指针B（数据类型相同才能赋值），这个操作，将两个指针指向同一块内存C。然后将指针A的值即存储在内存C中的数据修改为D，那么指针B的值，也变成B。因为，指针B和指针A的值，都是内存C中的数据。

将指针A占用的内存K中存储的值C这块内存区域里的数据F赋值给M，然后将A将内存K中的值修改为D，M所使用的内存中存储的数据仍然是F。

本项测试，勉强算通过了。我很不满意这种冗长、存在不理解语法的情况。

耗费时间2小时11分。过程如下：

1. 定位出了问题，全局变量单链表（变量单链表、函数if结构单链表等）头结点始终指向第一个结点。
2. 解决这个全局变量问题。
   1. 想在函数内使用static变量，可是不支持static变量使用malloc初始化。我不知道不初始化会怎样。
   2. 在`createFunction` 等函数内，使用完单链表头结点后，将头结点初始化。这导致已经被存储到AST的头结点成员值被初始化。
   3. 随机想到了尝试把单链表头结点指针指向的数据赋值给AST的单链表头结点。
      1. 这里是主要的时间消耗点。
      2. 使用`lldb` 打印变量时，若struct类变量是指针，需使用`->` 读取成员数据；若是非指针，需使用`.` 读取成员数据。这是我已经知道的。我困扰于p不能打印非指针中间变量，而只能打印非指针中间变量的成员，但是却能打印指针变量本身。我以为未能正确存储数据，胡乱尝试，才发现只能打印非指针中间变量的成员。

#### 解析while语句

**测试**

输入数据

```c
int main(int argc, char ef, double te) {
    int ab;
    int mf;
    char hi;
    if (true) {
        ab = 5;
        test_mf = 89;
        hi = 94;
    }else{
        _ab = 5;
        mf_ = 89;
        h_i = 94;
    }
    if (false) {
        ab_ = 5;
        mf_ = 89;
        hi_ = 94;
    }else{
        _ab__ = 5;
        mf___ = 89;
        h_i__ = 94;
    }
    while(true){
        ab=5;
        de=78;
    }
}##
```

lldb 命令

```text
 p *root->funcBody->funcStmtsListHead.next->funcStmtNode->thenExprNodeListHeader.next->next->expr->l
  p *root->funcBody->funcStmtsListHead.next->next->next->funcStmtNode->thenExprNodeListHeader.next->next->expr->l
```

时间消耗：47分。复制粘贴而已，新增一个函数，稍微修改bison文件就行。时间消耗点，测试，不应该消耗这么多时间，我也想不起为啥会用这么多时间，需要更细化地记录时间消耗。

#### do-while

写思路，时间消耗：5分。

代码修改：6分。

测试：

1. 报错，重复运行3分。
   1. 解决了问题，9分。修改错误用时几秒钟，发现问题用了剩余时间。也没啥好方法，运行了几次。没有发现。看bison文件，发现，新增的do\_while\_stmt没有加入整个体系。
   2. 测试通过：8分。

总共耗时：31分。测试和找问题消耗时间17分。写代码只有6分钟而已。

1. 修改lex文件。
2. 新增数据结构和创建结点的函数。
3. 修改bison文件。

**测试**

输入数据

```c
int main(int argc) {
    int ab;
    do{
        ab = 5;
        mf___ = 89;
        h_i__ = 94;
    }while (true);
}##
```

lldb命令

```text
 p *root->funcBody->funcStmtsListHead.next->funcStmtNode->thenExprNodeListHeader.next->expr->l
```

输入数据

```c
int main(int argc) {
    int ab;
    do{
        ab = 5;
        mf___ = 89;
        h_i__ = 94;
    }while(true);
    if (true) {
        ab = 5;
        test_mf = 89;
        hi = 94;
    }else{
        _ab = 5;
        mf_ = 89;
        h_i = 94;
    }
    if (false) {
        ab_ = 5;
        mf_ = 89;
        hi_ = 94;
    }else{
        _ab__ = 5;
        mf___ = 89;
        h_i__ = 94;
    }
    while(true){
        ab=5;
        de=78;
    }
}##
```

lldb命令

```text
p *root->funcBody->funcStmtsListHead.next->funcStmtNode->thenExprNodeListHeader.next->expr->l
p *root->funcBody->funcStmtsListHead.next->next->funcStmtNode->thenExprNodeListHeader.next->expr->l
p *root->funcBody->funcStmtsListHead.next->next->next->funcStmtNode->thenExprNodeListHeader.next->expr->l
p *root->funcBody->funcStmtsListHead.next->next->next->next->funcStmtNode->thenExprNodeListHeader.next->expr->l
```

```text
error: memory exhausted
```

#### for

规则有点难写，有点多，都是重复代码。

我想终止用bison解析语言结构了。目前我解析的是C语言代码，我要写的编译器是golang的。知道怎么用bison解析代码后，就可以开始解析go代码了。就算我把C语言的所有结构都解析出来了，也不会让我一两个小时就能解析出go代码，到时候我还得重来一次。而且，bison这个工具，没必要用它做太多练习。

我的目的，很重要的目的，是尽快写完比较完善的go编译器。时间很宝贵啊。

接下来，我会给目前的进度做个快照，封存。我会做下面的事情：

1. 根据目前的AST生成中间代码，或生成我期望的代码。
2. 用Cpp结合bison和lex来写代码。
3. 生成汇编代码，编译成机器语言，完成整个编译器流程。没必要等到解析语言结构的所有代码完成后再去打通整个流程。

#### 杂项

block不支持不带花括号的代码。暂未想到实现不带花括号的block的规则。

#### C语言struct“继承”

```c
#include <stdio.h>

//父结构体
struct father
{
    int f1;
    int f2;
};

//子结构体
struct son
{
    //子结构体里定义一个父结构体变量，必须放在子结构体里的第一位
    struct father fn;
    //子结构体的扩展变量
    int s1;
    int s2;
};

void test(struct son *t)
{
    //将子结构体指针强制转换成父结构体指针
    struct father *f = (struct father *)t;
    //打印原始值
    printf("f->f1 = %d\n",f->f1);
    printf("f->f2 = %d\n",f->f2);
    //修改原始值
    f->f1 = 30;
    f->f2 = 40;
}

int main(void)
{
    struct son s;
    s.fn.f1 = 10;
    s.fn.f2 = 20;

    test(&s);
    //打印修改后的值
    printf("s.fn.f1 = %d\n",s.fn.f1);
    printf("s.fn.f2 = %d\n",s.fn.f2);

    return 0;
}
```

son结构体里的第一个成员是father结构体类型的变量，son里的另外2个成员也是整形变量，这样，son结构体就好像继承了father结构体，并增加了2个成员。父结构体必须是第一个成员，才能模拟“继承”机制。

~~clion似乎不支持这个用法。~~

需要将子struct转为父struct，子struct才能使用父struct的成员。

### 错误

```text
(char *) $87 = 0x00000001003043f0 "intmainint argch_i__=94mf___=895ab=5true"
(lldb) n
tcc was compiled with optimization - stepping may behave oddly; variables may not be available.
Process 11465 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = step over
    frame #0: 0x0000000100002ad9 tcc`yyparse at y.tab.c:1520:3 [opt]
   1517         incorrect destructor might then be invoked immediately.  In the
   1518         case of YYERROR or YYBACKUP, subsequent parser actions might lead
   1519         to an incorrect destructor call or verbose syntax error message
-> 1520         before the lookahead is translated.  */
   1521      YY_SYMBOL_PRINT ("-> $$ =", YY_CAST (yysymbol_kind_t, yyr1[yyn]), &yyval, &yyloc);
   1522
   1523      YYPOPSTACK (yylen);
Target 0: (tcc) stopped.
```

```text
 warning: POSIX yacc reserves %type to nonterminals [-Wyacc]
```

```text
(char *) $32 = 0x0000000101004080 "truejqwepkjnm=89"
(lldb) n
Process 16694 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = step over
    frame #0: 0x00007fff67d12cc9 libdyld.dylib`start + 1
libdyld.dylib`start:
->  0x7fff67d12cc9 <+1>: movl   %eax, %edi
    0x7fff67d12ccb <+3>: callq  0x7fff67d2682e            ; symbol stub for: exit
    0x7fff67d12cd0 <+8>: hlt
    0x7fff67d12cd1 <+9>: nop
Target 0: (va_list_demo) stopped.
```

```text
fb1-5funcs.c:568:23: warning: passing 'const ExprNode *' (aka 'const struct exprNode *') to parameter of type 'ExprNode *'
      (aka 'struct exprNode *') discards qualifiers [-Wincompatible-pointer-types-discards-qualifiers]
    reverseLinkedList(&node->elseExprNodeListHeader);
```

```text
fb1-5.y:109:83: warning: result of comparison against a string literal is unspecified (use strncmp instead) [-Wstring-compare]
                                                { if((yyvsp[0].node)->stringValue != "``@@##``"){struct ast *variable = createVariable((yyv...
```

```text
error: Couldn't apply expression side effects : Couldn't dematerialize a result variable: couldn't read its memory
```

### 待解决问题

1. 写了反转链表的函数reverseLinkedList，但是，使用后，if结构会丢失元素。暂时搁置这个问题。
2. 不能解析含有等于号的表达式，例如：

   ```c
   int main(int argc){
       int a;
       int x;
       int str;
       str = "str=";
       a = 9;
       x = 7;
       printf(str, x);
   }
   ##
   ```

3. `str:%d\n` 怎么写bison规则？
4. `#include <stdio.h>` 在哪一步处理？是生成汇编代码时吗？在预处理时实现。怎么实现呢？
5. grep -R  '_int printf \(_' /\*  

### C语言转汇编

```text
# 将C代码转成汇编，并且用C代码变量来注释汇编
gcc -S -fverbose-asm test.c
```

#### 局部变量和返回

**整型**

```c
// c
int a = 5;
int b = 78;
return 0;
```

```text
// asm
subl    $16, %esp       #,    为啥是16？我发现，每次扩充局部变量空间，都是以16位单位，即：16、32、48。
movl    $5, -4(%ebp)    #, a
movl    $78, -8(%ebp)   #, b
movl    $0, %eax        #, D.2155 。return 0
```

**char**

```c
char c = 'h';
```

```text
subl    $16, %esp       #,
movb    $104, -1(%ebp)  #, c
movl    $0, %eax        #, D.2154
```

ebp的移动单位是byte，-1\(%ebp\) 的意思是，在ebp往低地址移动8个bit。

**字符串**

```c
char *str = "hello";
```

```text
   .section        .rodata
.LC0:
        .string "hello"
        .text
        .globl  main
        .type   main, @function
main:
.LFB0:
        .cfi_startproc
        pushl   %ebp    #
        .cfi_def_cfa_offset 8
        .cfi_offset 5, -8
        movl    %esp, %ebp      #,
        .cfi_def_cfa_register 5
        subl    $16, %esp       #,
        movl    $.LC0, -4(%ebp) #, str
        movl    $0, %eax        #, D.2154
        leave
        .cfi_restore 5
        .cfi_def_cfa 4, 4
        ret
        .cfi_endproc
```

汇编代码中，值为字符串的局部变量S，要写在开头。语法树是按顺序遍历的，先遇到函数名、参数等，然后再遇到S。次序与汇编代码的顺序矛盾，如何满足汇编代码的次序要求？

将汇编代码分成几部分，例如函数部分、rodata部分。遇到S，将之转为汇编代码，存入rodata部分。最后，将这些不同部分的代码组织成最终的汇编代码。

**简单函数调用**

and是与操作，这里是为了内存的对齐，有利于cpu的读取。

```text
andl    $-16, %esp      #,
subl    $16, %esp       #,
call    f       #
movl    $.LC0, 12(%esp) #, str
movl    $0, %eax        #, D.2161
```

声明变量，却不使用，有对应的汇编代码吗？

没有。

**具备变量和全局变量**

**字符串**

```text
char *str = "hi";
int main(){
    char *str;
    str = "hello";
}
```

```text
        .globl  str
        .section        .rodata
.LC0:
        .string "hi"
        .data
        .align 4
        .type   str, @object
        .size   str, 4
str:
        .long   .LC0
        .section        .rodata
.LC1:
        .string "hello"
```

**整型**

```c
int str = 900;
int main(){

        // f();
        int str;
        str  = 70;
        return 0;
}
```

```text
.globl  str
        .data
        .align 4
        .type   str, @object
        .size   str, 4
str:
        .long   900

 // some code
 subl    $16, %esp       #,
 movl    $70, -4(%ebp)   #, str
```

整型局部变量，直接在函数中赋值。

**if中的局部字符串变量**

```c
char *str = "900";
int main(){

        char *str;
        str  = "70";
        if(str > 0){
                char *a = "A";
                char *b = "D";
        }
        char *a = "AAAAA";
        return 0;
}
```

```text
        .globl  str
        .section        .rodata
.LC0:
        .string "900"
        .data
        .align 4
        .type   str, @object
        .size   str, 4
str:
        .long   .LC0
        .section        .rodata
.LC1:
        .string "70"
.LC2:
        .string "A"
.LC3:
        .string "D"
.LC4:
        .string "AAAAA"
        .text
        .globl  main
        .type   main, @function
main:
.LFB0:
        .cfi_startproc
        pushl   %ebp    #
        .cfi_def_cfa_offset 8
        .cfi_offset 5, -8
        movl    %esp, %ebp      #,
        .cfi_def_cfa_register 5
        subl    $16, %esp       #,
        movl    $.LC1, -4(%ebp) #, str
        cmpl    $0, -4(%ebp)    #, str
        je      .L2     #,
        movl    $.LC2, -8(%ebp) #, a
        movl    $.LC3, -12(%ebp)        #, b
.L2:
        movl    $.LC4, -16(%ebp)        #, a
        movl    $0, %eax        #, D.2167
        leave
        .cfi_restore 5
        .cfi_def_cfa 4, 4
        ret
        .cfi_endproc
```

1. 局部变量和全局变量的写法。
2. 同名局部变量，在汇编代码中，是不同的变量。

难道，我需要熟悉汇编代码的所有语法，才能将C语言翻译成汇编代码吗？

### 生成汇编代码

```c
#include <stdio.h>

int main(int argc){
    char *str;
    int x;
    str = "str=%d";
    x = 7;
    printf(str, x);
    return 0;
}
```

```text
        .section        .rodata
.LC0:
        .string "str=%d"
        .text
        .globl  main
        .type   main, @function
main:
.LFB0:
        .cfi_startproc
        pushl   %ebp    #
        .cfi_def_cfa_offset 8
        .cfi_offset 5, -8
        movl    %esp, %ebp      #,
        .cfi_def_cfa_register 5
        andl    $-16, %esp      #,
        subl    $32, %esp       #,
        movl    $.LC0, 28(%esp) #, str
        movl    $7, 24(%esp)    #, x
        movl    24(%esp), %eax  # x, tmp61
        movl    %eax, 4(%esp)   # tmp61,
        movl    28(%esp), %eax  # str, tmp62
        movl    %eax, (%esp)    # tmp62,
        call    printf  #
        movl    $0, %eax        #, D.2156
        leave
        .cfi_restore 5
        .cfi_def_cfa 4, 4
        ret
        .cfi_endproc
```

声明变量时初始化

```c
#include <stdio.h>

int main(int argc){
    char *str = "str=%d";
    int x = 7;
    //str = "str=%d";
    //x = 7;
    printf(str, x);
    return 0;
}
```

```text
        .section        .rodata
.LC0:
        .string "str=%d"
        .text
        .globl  main
        .type   main, @function
main:
.LFB0:
        .cfi_startproc
        pushl   %ebp    #
        .cfi_def_cfa_offset 8
        .cfi_offset 5, -8
        movl    %esp, %ebp      #,
        .cfi_def_cfa_register 5
        andl    $-16, %esp      #,
        subl    $32, %esp       #,
        movl    $.LC0, 28(%esp) #, str
        movl    $7, 24(%esp)    #, x
        movl    24(%esp), %eax  # x, tmp61
        movl    %eax, 4(%esp)   # tmp61,
        movl    28(%esp), %eax  # str, tmp62
        movl    %eax, (%esp)    # tmp62,
        call    printf  #
        movl    $0, %eax        #, D.2156
        leave
        .cfi_restore 5
        .cfi_def_cfa 4, 4
        ret
        .cfi_endproc
```

生成的汇编代码，与变量声明和变量赋值分开生成的汇编代码一样。

不理解这种汇编代码。函数调用，参数直接用压栈操作不就可以吗？

这种汇编代码，复杂又难以理解，我不生成这种汇编代码。

#### 生成汇编代码

```text
.section .data
str:
    .asciz  "hello,world:%d\n"
    len = . - str

.section .text
.global _start
_start:
    nop
    pushl $7
    pushl $str
    call printf
    movl $1, %eax
    int $0x80
```

这就是我要生成的汇编代码，分为三部分：\_start模板、call模板、变量。

\_start模板

```text
.section .text
.global _start
_start:
    nop
    # some code
    movl $1, %eax
    int $0x80
```

call 模板

```text
pushl $7
pushl $str
call printf
```

变量

```text
.section .data
str:
    .asciz  "hello,world:%d\n"
    len = . - str
```

在\_start模板中使用占位符，插入call模板，使用的C函数是？

> The printf\(\) family of functions produces output according to a format as described below. The printf\(\) and vprintf\(\) func- tions write output to stdout, the standard output stream; fprintf\(\) and vfprintf\(\) write output to the given output stream; dprintf\(\) and vdprintf\(\) write output to the given file descriptor; sprintf\(\), snprintf\(\), vsprintf\(\), and vsnprintf\(\) write to the character string str; and asprintf\(\) and vasprintf\(\) dynamically allocate a new string with malloc\(3\).

使用`sprintf`。c函数真奇怪，使用指针来获取结果，而不是直接返回。下面是使用`sprintf`的例子。

```c
#include <stdio.h>
#include <math.h>

int main()
{
   char str[80];

   sprintf(str, "Pi 的值 = %f", M_PI);
   puts(str);

   return(0);
}
```

```text
chugangdeMacBook-Pro:c-example cg$ gcc -o sprintf-demo sprintf-demo.c
chugangdeMacBook-Pro:c-example cg$ ./sprintf-demo
Pi 的值 = 3.141593
```

存储变量的哈希表variable\_hash\_table，也是个难点。

​ 这是简化后的场景。实际场景中，存在同名变量，这个哈希表不能满足要求。

我设计成这样：

```javascript
[
    ['str','char','hello'],    // 变量名，变量类型，变量汇编模板
]
```

`variable_hash_table` 是一个三维数组（是这样吗？），大数组的每个元素是三维数组，可以直接声明`arr[3]` ，大数组的长度不确定，我打算直接使用一个比较大的长度，而且，使用全局变量。

遍历变量链表时，创建哈希表，变量汇编模板中的变量值使用占位符。

遍历assignNode时，从哈希表中取出变量，若是字符串，替换变量汇编模板中的占位符；若是整型，直接存放整型值。

处理callStmtNode时，先反转实参链表，然后从实参链表中拿出变量名，从`variable_hash_table`中获取对应的变量，生成压栈汇编代码。

没有明确的思路啊，也不能很快想出明确的思路。在写CRUD时，理清了需求，我能够很容易从上至下将思路层层分解。面对现在的这个自己提的需求，为啥就做不到这点呢？

整个过程是这样的：

1. 遍历到main节点，生成\_start模板 startCode，调用函数部分使用占位符flag。
2. 遍历到变量链表，加入`variable_hash_table` 。
3. 遍历assignStmtNode链表，生成变量汇编模板 variableCode。
   1. 根据变量名，在`variable_hash_table` 查找该变量的变量类型。
      1. 若变量是字符串，生成变量汇编模板。
      2. 变量是整型，不处理。
4. 处理callNode，生成调用函数的汇编模板。
   1. 反转实参单链表。
   2. 遍历实参单链表，生成入栈汇编代码 pushCode。
   3. `call 函数名` callFunc。
   4. 拼接 pushCode + callFunc，生成callCode。
5. 生成完整的汇编代码。
   1. 使用callCode替换startCode中的flag。
   2. 拼接 startCode + variableCode。
   3. 使用我自定义的字符串拼接函数、sprintf都行。

时间消耗1个小时10分钟。

cg.s: Warning: end of file not at end of a line; newline inserted

解决：在最后一行末尾敲入一个enter键，解决了问题。

flex匹配字符串中的换行，费了很多时间才试验出来。查资料没用。

```text
[a-z_A-Z0-9:%=\\n]*  { yylval.strval = strdup(yytext);  return STR;  }
```

```c
int main(int argc){
    char *str;
    int x;
    str = "str2=%d\n";
    x = 18;
    printf(str, x);
    return 0;
}
##
```

### 生成汇编语言后

把高级语言转为汇编语言后，剩下的工作，只是使用汇编器来编译汇编代码生成可执行程序，没有多少工作量。

例如：

```text
#! /bin/bash

echo "filename:$0"

code=$1
echo "code:$1"


as -gstabs -o $code.o $code.s
ld -dynamic-linker /lib/ld-linux.so.2 -o $code -lc $code.o
```

这个文件在centos-i386容器中，文件名是c.sh。使用它编译汇编代码：c.sh filename，不加文件后缀。

最后两条命令：

```text
as -gstabs -o $code.o $code.s
ld -dynamic-linker /lib/ld-linux.so.2 -o $code -lc $code.o
```

不能在macOSX上执行，在centos-i386上可以正确执行。

而我的生成汇编代码的C代码，应该不能在centos-i386上执行，所以，我不能写一个“编译、生成可执行文件”的完整编译器模型了。

不过，只能生成的汇编代码，能够在i386系统上编译执行，就行了。

可以正式开始写golang编译器了。

参考资料

writing-your-own-toy-compiler

[https://gnuu.org/2009/09/18/writing-your-own-toy-compiler/](https://gnuu.org/2009/09/18/writing-your-own-toy-compiler/)

中文版

[https://www.cnblogs.com/linucos/archive/2012/09/26/2704387.html](https://www.cnblogs.com/linucos/archive/2012/09/26/2704387.html)

### 临时

Vim中显示不可见字符

`cat -A file`可以把文件中的所有可见的和不可见的字符都显示出来，

只需要`:set invlist`即可以将不可见的字符显示出来，例如，会以`^I`表示一个tab符，`$`表示一个回车符等。`:set nolist`可以回到正常的模式。

vim字符编码设置

vim的设置一般放在/etc/vimrc文件中，不过，建议不要修改它。可以修改~/.vimrc文件（默认不存在，可以自己新建一个），写入所希望的设置。

```text
:set encoding=utf-8
:set fileencodings=ucs-bom,utf-8,cp936
```

## centos : yacc&lex gcc cannot find -ll/-lfl

yum install flex flex-devel

