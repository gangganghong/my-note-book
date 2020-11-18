---
description: 先弄清楚怎么用AST生成某种代码，再用AST生成中间代码，避免一开始就去确定使用哪种格式的中间代码。
---

# 用AST生成新代码

## 思路

这个问题困扰了我很久，我看了几次cbc代码，仍不清楚怎么用ast生成新代码。

旧代码的所有元素已知，按理说，很容易生成新代码。我的疑惑是，人脑知道所有元素，一下子就组装成新代码了。可编程将元素组装成新代码，我却觉得不知道怎么做。遍历AST的时候，只能知道一个结点是不是函数结点，函数的所有其他元素还需要继续遍历该结点的子结点。如何在不知道所有子结点的情况下组成新代码？

按照AST顺序，是从下往上还是从上往下组装？

我想不明白，只能写代码试试了。

以下面的代码为例子：

```c
int main(int ab, int df){
    if(true){
        char you;
        char ok;
    }
}
```

设置这段代码的AST的根节点是root。root的子节点有：返回数据类型、函数名、函数参数、函数体。

返回数据类型、函数名，能不需要访问它们的子节点就获得值，能直接打印或放入新代码中。

函数参数，需要访问子节点，并且，不知道需要访问到第几层才能获得它们的值，该如何处理？

函数体更复杂，又该如何处理？

遍历二叉树、逆波兰、栈，哪个能用到这里？

不能用栈，因为，丢失了信息。两个参数节点存储到栈，我怎么知道这是两个参数而不是两个变量。

不知道能不能用逆波兰，不是很熟，应该不能用吧。因为，没有存储括号等。

AST是一颗树，遍历AST就是遍历树？用哪种遍历方法？

先序遍历：根左右

中序遍历：左根右

后序遍历：左右根

root的子节点按顺序排列是：返回数据类型、函数名、函数参数、函数体。遍历时要保持这个顺序，应使用先序遍历。

处理函数参数，根据nodeType识别出是函数参数列表节点后，遍历存储参数的单链表，获取每个节点的数据，组装成目标代码的函数参数字符串，作为返回结果。

假设处理if结构的函数是B，从if节点中获取con、then、elseBody。

con比较好办，con->l->stringValue。

then，包含多个表达式expr（简化为ab=5;）。elseBody，也一样。多个表达式存储在多表达式单链表中。

处理expr，left->value + nodeType + right->value。

这就是递归！我知道递归是什么，已经用过很多次递归了。为什么不知道怎么在这里套用递归呢？

这里的递归，包含很多特殊条件，上面只是简化版本。

递归的终止条件是：递归函数的参数是 IDENTIFIER 或 NUMBER。

递归函数的返回值是char *。

不管内部如何，只需认为递归函数返回的是需要的字符串。假设递归函数是generateCode(node)，那么，上面的函数表示为：

char *funcStr = generateCode(返回数据类型) + generateCode(函数名) + generateCode(函数参数) + generateCode(函数体)

又遇到问题。在ast中，返回数据类型、函数名、函数参数、函数体 不全是相同的数据类型，一个函数要求同一个参数的数据类型一致。

怎么办？

1. 将所有在bison规则中出现的节点包装成node类型，例如，函数参数列表包装成node类型，该node有一个成员变量是单链表，在generateCode中识别出是单链表后，然后处理单链表。
   1. 不打算用这种方式，感觉很杂乱。
2. generateCode作为总函数，不用递归，对不同的子节点，使用不同的递归函数。
   1. node类型节点：使用递归函数A。
   2. 单链表：遍历单链表，对它的每个节点按类型处理，在函数B中实现
      1. node类型，使用A。
      2. 非node类型，使用它的处理函数。我暂时未想到是什么。
      3. 单链表类型，使用B。

我只愿想到这里了。遍历AST问题解决了，生成目标代码字符串，简单。

<!--不需要完全想清楚了再写出来，这是开发笔记，在我没有思路的时候，就用写来帮助思考，不易分心-->

写代码



## 时间消耗

### 小结

主要时间消耗在调试上。没有困难的算法，仅仅是因为代码写得长了点，使用了递归。

只会使用lldb对函数设置断点，很不方便。



#### 错误类型

1. 逻辑遗漏。有些节点没有处理。
2. 逻辑错误。此处和彼处不吻合。就是指con节点。简化版con是true这样的标识符，所以我改造了con，使它有l、r,但是无numString，可是con的nodeType是STR_NODE_TYPE。遍历节点时，遇到STR_NODE_TYPE会读取numString。有错误不要紧，但是要有方法顺序定位问题。像这样单步调试，实在耗费时间。精神不好时，恐怕还找不出错误。
3. 封装函数后，没有把所有相关地方都修改正确。
4. 粗心。单独拿出来，一眼就能看出的错误。和其他代码混合在一起了，就很难定位问题。

#### 障碍

1.C语言常用用法。

	1.	拼接两个字符串，拼接多个字符串。strcat、strcpy等函数用法。
 	2.	函数返回字符串。参数指针、像PHP那样直接返回函数内的变量，两者方法有时候能用，有时候不能用。目前，我仍不知道其中原因。

### 具体情况

1. 7分。

   1. 用web版gitbook新建文件，避免我在本地创建文件，能不必手工把汉字转拼音，不用写文章目录。
   2. 使用web版做这个事很耗费时间，以后可以自己写个小工具，自动化这个事情。

2. 1小时10分。

   1. 构思如何遍历AST生成目标代码。

3. 实现。

   1. 33分。将nodeType改成使用enum。跑demo，复制粘贴代码。

   2. 10分。使用“继承”struct引发的错误，暂时改不好了，还原了，仍使用冗余成员。

   3. 23分。多次改动“继承”struct，仍不能消除错误。

      1. 到目前为止，受阻于C语言语法问题，还未到业务逻辑这一步。
      2. 错误快照分支：v1-extends-struct，文件：/Users/cg/data/code/study-compiler-java/study/flex-and-bison/flex/golang/fb1-5funcs.c
      3. 在这个语法上，浪费这么多时间，错误原因未知。我无清晰思路，靠直觉调试，下次不可如此。

   4. 换用union试试。

      1. 14分。复制粘贴代码。

      2. 21分。写完遍历node节点函数traverseNode。复制粘贴，繁琐，雷同，稍有不慎，会遗漏或多加点什么。

      3. 遍历单链表funcStmtNode

         1. 10分。写了一点逻辑，然后受阻于C语言连接字符串，没有PHP、JS、C++那么方便。

         2. 24分。C语言连接字符串。常见用法，不了解。

            1. 两个已知字符串连接。

            2. 一个循环里，不确定多少个字符串连接，如下：

            3. ```c
               char *codeStr = "";
               while (cur != NULL) {
                 char *temp = traverseNode(cur->linkedListNode);
                 cur = cur->next;
               }
               ```

            4. ```c
               // 正确性未知
               char *codeStr = "";
                       while (cur != NULL) {
                           char *temp = traverseNode(cur->linkedListNode);
                           // todo 这句，能否将一个字符串赋值给另外一个？
                           char *oldCodeStr = codeStr;
                           codeStr = (char *)malloc(sizeof(char) * strlen(temp) + strlen(oldCodeStr));
                           strcpy(codeStr, oldCodeStr);
                           strcpy(codeStr, temp);
                           cur = cur->next;
                       }
               ```

      4. 7分。遍历单链表funcStmtNode、exprNode、paramNode、funcVariableNode。复制粘贴。

      5. 43分。初步测试。遇到函数返回字符串问题。字符串是内部变量，不能像PHP那样由函数返回。我改成了由函数传入指针参数来获取返回数据。由于大量重复代码，需要修改很多地方。

      6. 28分。测试。无结果，断点调试，啥也看不出来。重看代码，发现有很多nodeType没有修改过来，漏掉了一些重要节点的遍历。

      7. 24分。检查代码，漏掉了太多重要节点（元素）处理逻辑，复制粘贴，补上它们。写之前，应该考虑全面。

      8. 21分。发现，多次使用strcpy拼接字符串的代码不能实现预期目的。代码如下：

         1. ```c
            #include <stdio.h>
            #include <stdlib.h>
            #include <string.h>
            
            int main(){
                    char *codeStr = " world!";
                    char *str = "hello";
                    char *oldCodeStr = codeStr;
                    printf("oldCodeStr = %s\n", oldCodeStr);
                    codeStr = (char *) malloc(strlen(str) + strlen(oldCodeStr));
                    strcpy(codeStr, oldCodeStr);
                    strcpy(codeStr, str);
                    printf("codeStr = %s\n", codeStr);
                    return 0;
            }
            ```

            执行结果：

            ```shell
            // 执行结果
            chugangdeMacBook-Pro:c-example cg$ ./a.out
            oldCodeStr =  world!
            codeStr = hello
            ```

      9. 13分。重新找拼接多个字符串的方法。找到了。

         ```c
         // /Users/cg/data/code/c-example/strcat_demo.c
         #include <stdio.h>
         #include <stdlib.h>
         #include <string.h>
         
         int main(){
                 char *codeStr = " world!";
                 char *str = "hello";
                 char *oldCodeStr = codeStr;
                 printf("oldCodeStr = %s\n", oldCodeStr);
                 codeStr = (char *) malloc(strlen(str) + strlen(oldCodeStr));
                 strcat(codeStr, oldCodeStr);
                 strcat(codeStr, str);
                 printf("codeStr = %s\n", codeStr);
                 return 0;
         }
         ```

         执行结果：

         ```shell
         chugangdeMacBook-Pro:c-example cg$ ./a.out
         oldCodeStr =  world!
         codeStr =  world!hello
         ```

      10. 47分。找拼接多个字符串的C语言代码。

          ```c
          // /Users/cg/data/code/c-example/stdarg_demo.c
          #include <stdio.h>
          #include <stdarg.h>
          #include <stdlib.h>
          #include <string.h>
          
          double average(int num,...)
          {
          
              va_list valist;
              double sum = 0.0;
              int i;
          
              /* 为 num 个参数初始化 valist */
              va_start(valist, num);
          
              /* 访问所有赋给 valist 的参数 */
              for (i = 0; i < num; i++)
              {
                 sum += va_arg(valist, int);
              }
              /* 清理为 valist 保留的内存 */
              va_end(valist);
          
              return sum/num;
          }
          
          // 为什么，这个函数能够使用局部变量返回字符串？
          char *contactStr(int num,...)
          {
              va_list valist;
              int i;
              char *codeStr = "";
          
              /* 为 num 个参数初始化 valist */
              va_start(valist, num);
          
              /* 访问所有赋给 valist 的参数 */
              /* 访问所有赋给 valist 的参数 */
              for (i = 0; i < num; i++)
              {
                 char *str = va_arg(valist, char *);
                 printf("str%d = %s\n", i, str);
                 char *oldCodeStr = codeStr;
                 printf("old = %s\n", oldCodeStr);
                 codeStr = (char *)malloc(strlen(str) + strlen(oldCodeStr));
                 strcat(codeStr, oldCodeStr);
                 printf("codeStr1 = %s\n", codeStr);
                 strcat(codeStr, str);
                 printf("codeStr2 = %s\n", codeStr);
              }
              /* 清理为 valist 保留的内存 */
              va_end(valist);
          
              return codeStr;
          }
          
          int main()
          {
             printf("Average of 2, 3, 4, 5 = %f\n", average(4, 2,3,4,5));
             printf("Average of 5, 10, 15 = %f\n", average(3, 5,10,15));
          
             char *str1 = "hello";
             char *str2 = " world";
             char *str3 = "hi";
             char *codeStr = contactStr(3, str1, str2, str3);
             printf("codeStr = %s\n", codeStr);
          
             return 0;
          }
          ```

          执行结果：

          ```shell
          chugangdeMacBook-Pro:c-example cg$ gcc stdarg_demo.c
          chugangdeMacBook-Pro:c-example cg$ ./a.out
          Average of 2, 3, 4, 5 = 3.500000
          Average of 5, 10, 15 = 10.000000
          str0 = hello
          old =
          codeStr1 =
          codeStr2 = hello
          str1 =  world
          old = hello
          codeStr1 = hello
          codeStr2 = hello world
          str2 = hi
          old = hello world
          codeStr1 = hello world
          codeStr2 = hello worldhi
          codeStr = hello worldhi
          ```

          为啥耗费这么多时间？

          1. 没有特别的困难，都是常规调试，在其他地方，不能用局部变量作为函数的返回值来返回字符串，需要指针。在这里，却反过来了。面对这个结果，我不得不把代码改成用局部变量返回字符串的用法。我认为，时间消耗这么多，是因为先用了错误代码，然后再修改，时间花在了修改代码、重新运行上。若需要绝对精确的知道什么事件花了时间，只能统计更细粒度的事项。
          2. 如果我对C语言拼接多个字符串等常用功能很熟悉，这40多分钟，可以节约下来。

      11. 修改项目代码中拼接多个字符串的方法。

          1. 11分。复制粘贴代码，修改完成。

          2. 7分。测试。每写完一个就手工测试。方法是运行第一个调用的函数。我一直以来都是这么做的。

             1. 断点调试，递归，结果肯定有问题，void traverseNode(struct ast *node, char *codeStr) 和 void traverseLinkedList(SINGLE_LINKED_LIST_NODE_HEADER header, int headerType, char *codeStr) 没有产生拼接后的codeStr。执行了很多层递归，codeStr仍然只是 (char *) $22 = 0x00000001003041c0 "intmain"。

          3. 调试。

             1. 6分（只是想出思路的时间，下面是各项花费的时间）。调试思路是什么？

                1. 3分。弄清楚，为啥codeStr能正确拼接出"intmain"。
                   1. 没有运行代码。只是想了一会，原因是，“intmain"是在没有递归的情况下拼接的。
                2. 要被拼接的字符串，是否都被一个不少地传到contactStrBetter。
                3. 在递归过程中，codeStr有没有发生变化？
                   1. 2分。在void traverseNode(struct ast *node, char *codeStr) 内查看FUNC_NODE_TYPE分支，在traverseLinkedList 中查看PARAM_HEADER分支。
                   2. 14分。执行第1步。
                      1. codeStr正常。
                      2. 发现了新问题，traverseNode中漏掉了对函数参数、函数变量的处理。
                      3. 为啥会花这么多时间？没有难点，全是断点，然后核对数据。在看数据时稍微需要看着代码来确定数据是否正常。这个速度，基本是正常的。花这么多时间，只能说明，调试这个事情，本来就占用时间。看一篇无难度的公众号文章，也需要七八分钟呢。要想不花这个时间，只能采取更高效的测试方法。
                4. void traverseNode(struct ast *node, char *codeStr) 这种方式，究竟能不能获得拼接后的codeStr？

             2. 在traverseNode中补充对函数参数、函数变量的处理。

                1. 10分。改好。复制粘贴。记忆力是靠不住的，需写详细注释。最快可以用5分钟完成。

             3. 调试。

                1. 思路：在contactStrBetter设置断点，查看数据中是否出现函数参数。

                2. 10分。createIfNode，对else为NULL的处理。改了好几次依靠编译器提示才改正确，完全是粗心导致！

                3. 4分。Continue.

                   1. contactStrBetter正常。
                   2. 每个节点都被正常传进来了。
                   3. 异常，contactStrBetter 接收到的codeStr总是intmain。我断定：是遍历函数使用参数获取返回结果出了问题。

                4. 再修改void traverseNode(struct ast *node, char *codeStr) void traverseNode(struct ast *node, char *codeStr)  为char *traverseNode(struct ast *node) 

                   1. 22分。全部工作量都是复制粘贴修改。如果又遇到不能返回局部变量的问题。这22分钟，再加上改回去的几十分钟，将被全部浪费！更好的做法，先确定新方案是否行得通。之前试过，不行，现在，我想再试一次，但是完全无任何依据预测，再试一次可能会正确。
                   2. 运行第4步修改后的代码。
                      1. 2分。编译。无之前的返回局部变量警告。原因不明。
                      2. 发现了因修改方案而遗漏的地方。这有点无法避免。
                      3. 5分。运行，测试。
                         1. 思路，仍然是观察contactStrBetter内的数据。
                         2. 正常。但是遇到不明原因段错误。

                5. 运行newCode，遇到不明原因段错误。

                   1. 5分。打算怎么办？

                      1. 在入口设置断点，一步步往下调试，看看问题出在哪一步。
                         1. 这是递归，执行步骤非常多，此法不可取。
                      2. 我只会使用lldb在函数设置断点，我的代码粒度太粗，无法惊喜地用二分法设置断点。
                      3. 先看一次代码。
                      4. 简化输入数据。

                   2. 6分。看代码。

                      1. 懒得细看，看了一点不想看了。

                   3. 输入数据，是一个函数，由多个节点组成，我在traverseNode设置断点，观察是否每个函数节点都正确处理了。

                      1. 18分。测试方案。

                         1. 

                         ```basic
                         FUNC_NODE_TYPE
                         STR_NODE_TYPE
                         PARAM_NODE_TYPE
                         BLOCK_NODE_TYPE
                         VARIABLE_NODE_TYPE
                         IF_NODE_TYPE
                         CON_NODE_TYPE
                         EXPR_NODE_TYPE
                         ```

                         

                         2. 18分。在整理的过程中，我发现遗漏了处理ExprNode的逻辑，补上了，复制粘贴的拼接字符串代码。

                      2. 测试。

                         1. 9分。使用lldb的c命令，观察依次处理的节点类型：

                            1. STR_NODE_TYPE 没有 在 PARAM_NODE_TYPE 前出现，没报错。函数类型和函数名在FUNC_NODE_TYPE时就已经直接获取了，所以，这个顺序是正常的。

                            2. BLOCK_NODE_TYPE出现后，继续执行c，报错。原因是执行`char *codeStr1 = traverseNode(node->con);` ，BLOCK_NODE_TYPE节点无con。

                               报错信息如下：

                            3. ```shell
                               (lldb) p node->nodeType
                               error: Couldn't apply expression side effects : Couldn't dematerialize a result variable: couldn't read its memory
                               ```

                         2. 8分。修改后继续测试。找到了出错大概地方。

                            1. CON_NODE_TYPE没有紧跟IF_NODE_TYPE出现。
                            2. STR_NODE_TYPE 之后报错。

                         3. 聚焦if结构、STR_NODE_TYPE测试。

                            1. 3分。看代码看不出问题。

                            2. 6分。把这块代码封装成函数traverseIfNode，方便直接在这里设置断点。使用clion完成。若能直接用clion设置断点，无需这样做就能断点。

                            3. 5分。观察 traverseIfNode。找到了出错点，traverseNode(con)

                               ```c
                               // if(con)中的con节点
                                   if (node->nodeType == CON_NODE_TYPE) {
                                       traverseNode(node->con);
                                       return codeStr;
                                   }
                               ```

                               con是一个表达式，要么把它转发给表达式逻辑处理，要么，不再把con当作一个单独的结点，而直接当成表达式结点。

                               暂时把它转发给表达式逻辑处理吧。

                            4. 10分。仍未解决问题，但发现了新的错误，ab = 5 + 4 类型的表达式可能不会被处理。

                            5. 12分。修复4中的问题。

                               1. 封装函数traverseExprNode、isExprNode，便于设置断点。

                            6. Continue.

                               1. 设置断点traverseExprNode、isExprNode，观察。

                                  1. 4分。封装函数时遗漏了一些地方，不匹配，fix。

                                  2. 3分。观察。

                                     1. 直接就报错了。一步也不能运行。原因未知。

                                        ```shell
                                        Process 12972 stopped
                                        * thread #1, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=1, address=0x0)
                                            frame #0: 0x00007fff72acfe52 libsystem_platform.dylib`_platform_strlen + 18
                                        libsystem_platform.dylib`_platform_strlen:
                                        ->  0x7fff72acfe52 <+18>: pcmpeqb (%rdi), %xmm0
                                            0x7fff72acfe56 <+22>: pmovmskb %xmm0, %esi
                                            0x7fff72acfe5a <+26>: andq   $0xf, %rcx
                                            0x7fff72acfe5e <+30>: orq    $-0x1, %rax
                                        Target 0: (tcc) stopped.
                                        ```

                               2. 设置断点：traverseExprNode、isExprNode、traverseNode。在traverseNode内单步调试。

                                  1. 6分。C语言的bool值问题，我用enum模拟实现。

                                  2. 11分。Continue.找到了错误原因。代码如下：

                                     ```c
                                     struct ast *createCon(struct ast *con) {
                                         struct ast *node = malloc(sizeof(struct ast));
                                         node->nodeType = con->nodeType;
                                         node->con = con;
                                         if (con->nodeType == STR_NODE_TYPE || con->nodeType == NUM_NODE_TYPE) {
                                     //        无法匹配 ab = 5 + 4 这样的表达式
                                     // todo 后期，我要删除这个con节点，把它当普通的表达式
                                     //        node->nodeType = EXPR_NODE_TYPE;
                                             node->l = con;
                                         } else {
                                             node->l = con->l;
                                         }
                                         node->r = con->r;
                                     
                                         return node;
                                     }
                                     ```

                                     ```c
                                     // 标识符
                                         if (node->nodeType == STR_NODE_TYPE) {
                                             char *str = node->stringValue;
                                             char *oldCodeStr = codeStr;
                                             codeStr = contactStrBetter(2, oldCodeStr, str);
                                             return codeStr;
                                         }
                                     ```

                                     true这样的表达式被识别为con类型节点，可是它没有设置stringValue值。str、oldCodeStr都是空值，至于为啥会报错：

                                     ```c
                                     Process 13318 stopped
                                     * thread #1, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=1, address=0x0)
                                         frame #0: 0x00007fff72acfe52 libsystem_platform.dylib`_platform_strlen + 18
                                     libsystem_platform.dylib`_platform_strlen:
                                     ->  0x7fff72acfe52 <+18>: pcmpeqb (%rdi), %xmm0
                                         0x7fff72acfe56 <+22>: pmovmskb %xmm0, %esi
                                         0x7fff72acfe5a <+26>: andq   $0xf, %rcx
                                         0x7fff72acfe5e <+30>: orq    $-0x1, %rax
                                     Target 0: (tcc) stopped.
                                     ```

                                     我不确定原因。猜测如下：

                                     ```c
                                     char *oldCodeStr = codeStr;
                                     //        printf("old = %s\n", oldCodeStr);
                                             codeStr = (char *) malloc(strlen(str) + strlen(oldCodeStr));
                                             strcat(codeStr, oldCodeStr);
                                     //        printf("codeStr1 = %s\n", codeStr);
                                             strcat(codeStr, str);
                                     ```

                                     malloc不能分配空内存，strcat不能连接到空内存字符串。

                               3.  continue.

                                  1. 18分。单步观察，找到了出错的地方：

                                     错误信息：

                                     ```shell
                                     Process 13691 stopped
                                     * thread #1, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=1, address=0x0)
                                         frame #0: 0x00007fff72acfe52 libsystem_platform.dylib`_platform_strlen + 18
                                     libsystem_platform.dylib`_platform_strlen:
                                     ->  0x7fff72acfe52 <+18>: pcmpeqb (%rdi), %xmm0
                                         0x7fff72acfe56 <+22>: pmovmskb %xmm0, %esi
                                         0x7fff72acfe5a <+26>: andq   $0xf, %rcx
                                         0x7fff72acfe5e <+30>: orq    $-0x1, %rax
                                     Target 0: (tcc) stopped.
                                     ```

                                     

                                     ```c
                                     if (node->nodeType == BLOCK_NODE_TYPE) {
                                             SINGLE_LINKED_LIST_NODE_HEADER header;
                                             header.funcVariableNode = node->funcVariableListHead;
                                             char *codeStr2 = traverseLinkedList(header, FUNC_VARIABLE_HEADER);
                                             header.funcStmtNode = node->funcStmtsListHead;
                                             char *codeStr3 = traverseLinkedList(header, FUNC_STMT_HEADER);
                                     
                                             char *oldCodeStr = codeStr;
                                             codeStr = contactStrBetter(4, oldCodeStr, codeStr2, codeStr3);
                                     
                                             return codeStr;
                                         }
                                     ```

                                     contactStrBetter的第一个参数，是后面参数的个数，只有三个参数，却写了4。这完全是粗心。

                                     这种是普通逻辑。像Raft协议那些，我根本无法知道是否正确。

                                     其他的全部正常，我看到打印出来的旧代码（输入数据）的所有构成元素了，虽然不规范地连在一起，但是，先后顺序没有问题。

                                     ```shell
                                     (lldb) p codeStr2
                                     (char *) $41 = 0x0000000100305d90 "int abint mfchar hi"
                                     (lldb) p codeStr3
                                     (char *) $42 = 0x00000001007041d0 "truehi=94test_mf=89ab=5"
                                     ```

                                  2. 9分。Continue.全部正常了。输出信息：

                                     ```shell
                                     chugangdeMacBook-Pro:golang cg$ ./tcc
                                     int main(int argc, char ef, double te) {
                                         int ab;
                                         int mf;
                                         char hi;
                                         if (true) {
                                             ab = 5;
                                             test_mf = 89;
                                             hi = 94;
                                         }
                                     }##
                                     intmainint argcchar efdouble teint abint mfchar hitruehi=94test_mf=89ab=5
                                     ```

                                     3. 有问题的地方。单链表的节点顺序与原始代码的顺序是相反的。不知道对编译程序有没有影响。不过，就算是单链表，我也可以反转它。

                                     

          4. 其他。

             1. 尝到冗余代码的苦果了。需修改多个地方的代码，复制粘贴。
             2. 为何写冗余？直观，不需要动脑筋。
             3. C语言函数返回字符串，有时不能用局部变量返回。