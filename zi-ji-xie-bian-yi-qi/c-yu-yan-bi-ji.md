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

chugangdeMacBook-Pro:Downloads cg$ tar -jxvf CUnit-2.1-3.tar.bz2 tar: Error opening archive: Unrecognized archive format

tar -jxvf CUnit-2.1-3.bz2

chugangdeMacBook-Pro:CUnit cg$ make cd .. && /Library/Developer/CommandLineTools/usr/bin/make am--refresh \[../aclocal.m4\] Error 2

CUnit\*\# aclocal

automake: error: 'configure.ac' is required

CUnit 不是一个好工具，同样的下载链接，下载到不同的代码，如何使用的文档，没找到。博客上找到的教程，错误百出。

放弃使用它。

耗费时间：1小时15分。

sudo cp ./lib/libgtest\*.a /usr/local/lib

### c++中的nullptr时报错：Use of undeclares identifier 'nullptr'

编译时加flag： --std=c++11

C代码用g++编译，出现特别多不理解的错误，而用gcc编译是没问题的。

```text
                ^
```

../fb1-5funcs.c:643:10: error: 'operator=' cannot be the name of a variable or data member char operator = nodeType; ^ ../fb1-5funcs.c:643:20: error: expected ';' at end of declaration char operator = nodeType; ^ ;

放弃使用gtest测试c了。

p variableHashTable-&gt;variableTable\[0\]-&gt;name

\(lldb\) p strcmp\(codeStrTable-&gt;funcCodeTable\[0\].funcName, funcName\) error: :1:1: 'strcmp' has unknown return type; cast the call to its declared return type strcmp\(codeStrTable-&gt;funcCodeTable\[0\].funcName, funcName\) ^~~~~~~~~~~~~~~~~ \(lldb\) p \(int\)strcmp\(codeStrTable-&gt;funcCodeTable\[0\].funcName, funcName\) \(int\) $14 = 0

## 位域

```c
#include <stdio.h>
#include <string.h>
 
/* 定义简单的结构 */
struct
{
  unsigned int widthValidated;
  unsigned int heightValidated;
} status1;
 
/* 定义位域结构 */
struct
{
  unsigned int widthValidated : 1;
  unsigned int heightValidated : 1;
} status2;
 
int main( )
{
   printf( "Memory size occupied by status1 : %d\n", sizeof(status1));
   printf( "Memory size occupied by status2 : %d\n", sizeof(status2));
 
   return 0;
}
```

```shell
bit-field.c:20:54: warning: format specifies type 'int' but the argument has type
      'unsigned long' [-Wformat]
   printf( "Memory size occupied by status1 : %d\n", sizeof(status1));
                                              ~~     ^~~~~~~~~~~~~~~
                                              %lu
bit-field.c:21:54: warning: format specifies type 'int' but the argument has type
      'unsigned long' [-Wformat]
   printf( "Memory size occupied by status2 : %d\n", sizeof(status2));
                                              ~~     ^~~~~~~~~~~~~~~
                                              %lu
```

解决：把 `printf( "Memory size occupied by status1 : %d\n", sizeof(status1));` 改成 `printf( "Memory size occupied by status1 : %lu\n", sizeof(status1));` 。

```c
#include <stdio.h>
#include <string.h>
 
struct
{
  unsigned int age : 3;
} Age;
 
int main( )
{
   Age.age = 4;
   printf( "Sizeof( Age ) : %d\n", sizeof(Age) );
   printf( "Age.age : %d\n", Age.age );
 
   Age.age = 7;
   printf( "Age.age : %d\n", Age.age );
 
   Age.age = 8; // 二进制表示为 1000 有四位，超出
   printf( "Age.age : %d\n", Age.age );
 
   return 0;
}
```

```shell
chugangdeMacBook-Pro:c-example cg$ gcc -o bit-field2 bit-field2.c
bit-field2.c:12:36: warning: format specifies type 'int' but the argument has type
      'unsigned long' [-Wformat]
   printf( "Sizeof( Age ) : %d\n", sizeof(Age) );
                            ~~     ^~~~~~~~~~~
                            %lu
bit-field2.c:18:12: warning: implicit truncation from 'int' to bit-field changes value from 8 to
      0 [-Wbitfield-constant-conversion]
   Age.age = 8; // 二进制表示为 1000 有四位，超出
           ^ ~
2 warnings generated.
```

修改正确后的代码：

```shell
chugangdeMacBook-Pro:c-example cg$ ./bit-field2
Sizeof( Age ) : 4
Age.age : 4
Age.age : 7
Age.age : 6
```

如果程序的结构中包含多个开关量，只有 TRUE/FALSE 变量，如下：

```c
struct
{
  unsigned int widthValidated;
  unsigned int heightValidated;
} status;
```

这种结构需要 8 字节的内存空间，但在实际上，在每个变量中，我们只存储 0 或 1。在这种情况下，C 语言提供了一种更好的利用内存空间的方式。如果您在结构内使用这样的变量，您可以定义变量的宽度来告诉编译器，您将只使用这些字节。例如，上面的结构可以重写成：

```c
struct
{
  unsigned int widthValidated : 1;
  unsigned int heightValidated : 1;
} status;
```

现在，上面的结构中，status 变量将占用 4 个字节的内存空间，但是只有 2 位被用来存储值。如果您用了 32 个变量，每一个变量宽度为 1 位，那么 status 结构将使用 4 个字节，但只要您再多用一个变量，如果使用了 33 个变量，那么它将分配内存的下一段来存储第 33 个变量，这个时候就开始使用 8 个字节。

"status 变量将占用 4 个字节的内存空间，但是只有 2 位被用来存储值。",不理解前半句。

## 打印内存地址

```c
#include <stdio.h>

int main(int argc, char *argv[])
{
    char jqwepkjnm  = 61;
    // todo 把字符转为字符串。正确性未知。
    char operatorStr[2];
    operatorStr[0] = jqwepkjnm;
    operatorStr[1] = '\0';
    printf("operator = %c\n", jqwepkjnm);
    printf("operatorStr = %s\n", operatorStr);
    printf("address of jqwepkjnm:%02X\n", &jqwepkjnm);
    int a;
    printf("address of a:%02X\n", &a);
    return 0;
}
```

```shell
chugangdeMacBook-Pro:c-example cg$ gcc -o char_demo char_demo.c
char_demo.c:12:43: warning: format specifies type 'unsigned int' but the argument has type 'char *' [-Wformat]
    printf("address of jqwepkjnm:%02X\n", &jqwepkjnm);
                                 ~~~~     ^~~~~~~~~~
                                 %2s
char_demo.c:14:35: warning: format specifies type 'unsigned int' but the argument has type 'int *' [-Wformat]
    printf("address of a:%02X\n", &a);
                         ~~~~     ^~
2 warnings generated.
chugangdeMacBook-Pro:c-example cg$ ./char_demo
operator = =
operatorStr = =
address of jqwepkjnm:E436A6FF
address of a:E436A6F8
```

如何才能没有警告地打印内存地址？