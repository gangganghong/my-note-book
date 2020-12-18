# C语言笔记

## 拼接字符串

```c
# file:/Users/cg/data/code/c-example/sprintf-demo.c
# 编译：gcc -g -o sprintf-demo sprintf-demo.c
# 使用 -g ，能在使用lldb的时候显示C代码而不是汇编代码。

#include <stdio.h>
#include <math.h>
#include <stdlib.h>
#include <string.h>
int main()
{
   char str[4];
   sprintf(str, "=%d", 12);
   puts(str);

   char *str2 = (char *)malloc(sizeof(char) * 20);
   str2 = "hello,world";
   puts(str2);

   //char *tpl = "hello,%s\n";
   // char *tpl = (char *)malloc(sizeof(char)*200);
   char *tpl = "hello,%s\n"; // 能直接声明并赋值。
  // 这是不是精确确定str3即sprintf的第一个参数的空间的方法？如果申请了很多空间，会不会浪费？函数结束后，局部变量空间会释放吧？
  // 即使会释放，诸多函数都浪费内存，一起执行时，内存也不够了。
  // 我没想到好方法，只能申请更多空间了。
   char *str3 = (char *)malloc(strlen(tpl) + sizeof(char) * 6);
  // char *str3; 会出错，我不知道原因。
   sprintf(str3, tpl, "world");
   puts(str3);
   return(0);
}
```

## 调用系统命令

```c
# file:/Users/cg/data/code/c-example/system-demo.c
# 编译：gcc -g -o system-demo system-demo.c
# 使用 -g ，能在使用lldb的时候显示C代码而不是汇编代码。

#include <stdio.h>
#include <string.h>
#include<stdlib.h>
 
int main ()
{
   char command[50];
 
   strcpy( command, "ls -l" );
   system(command);
 
   return(0);
}
```





## 单元测试

大量时间花在测试上，C语言中的错误，报错信息含糊，提供的有用信息少，不知如何调试。例如

`Segmentation fault: 11` 。

chugangdeMacBook-Pro:Downloads cg$ tar -jxvf CUnit-2.1-3.tar.bz2
tar: Error opening archive: Unrecognized archive format





tar -jxvf CUnit-2.1-3.bz2

chugangdeMacBook-Pro:CUnit cg$ make
cd .. && /Library/Developer/CommandLineTools/usr/bin/make  am--refresh
	 [../aclocal.m4] Error 2



CUnit*# aclocal



automake: error: 'configure.ac' is required

CUnit 不是一个好工具，同样的下载链接，下载到不同的代码，如何使用的文档，没找到。博客上找到的教程，错误百出。

放弃使用它。

耗费时间：1小时15分。

sudo cp ./lib/libgtest*.a  /usr/local/lib



### c++中的nullptr时报错：Use of undeclares identifier 'nullptr'

编译时加flag：　--std=c++11

C代码用g++编译，出现特别多不理解的错误，而用gcc编译是没问题的。

                    ^
../fb1-5funcs.c:643:10: error: 'operator=' cannot be the name of a variable or data member
    char operator = nodeType;
         ^
../fb1-5funcs.c:643:20: error: expected ';' at end of declaration
    char operator = nodeType;
                   ^
                   ;

放弃使用gtest测试c了。



p variableHashTable->variableTable[0]->name

(lldb) p strcmp(codeStrTable->funcCodeTable[0].funcName, funcName)
error: <user expression 18>:1:1: 'strcmp' has unknown return type; cast the call to its declared return type
strcmp(codeStrTable->funcCodeTable[0].funcName, funcName)
^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
(lldb) p (int)strcmp(codeStrTable->funcCodeTable[0].funcName, funcName)
(int) $14 = 0

