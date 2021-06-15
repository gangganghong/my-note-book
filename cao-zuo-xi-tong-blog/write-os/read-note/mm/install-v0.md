# 安装应用程序

## v1

用C代码写一个小程序，使用的库全部是自己编写的。这个库，把那些库文件打包成一个文件，然后，编译新写的程序时，和打包成的库文件一起编译。

需要写一个start.asm，这个文件中的汇编函数是`_start`，在这个函数中调用main函数。

把所有编译好的应用程序打包成一个`tar`文件。

使用dd等命令把`tar`文件写入到硬盘中。在初始化文件系统时，解压`tar`文件，并且把解压出来的应用程序写入硬盘中。

> 难道，我要把那些dd命令都默写出来吗？

## v2

怎么为自己的操作系统编写应用程序？

### 创建CRT

把应用程序会使用的库文件编译成`.o`文件，然后打包到一个文件中。

使用下面的命令完成：

```shell
gcc -o lib/string.o lib/string.c -c -fno-builtin
gcc -o lib/fork.o lib/fork.c -c -fno-builtin
ar rcs my_crt.a lib/*.o
```



### 编写应用程序

C代码

```c
// pwd.c
#include "stdio.h"

int main(int argc, char * argv[])
{
  		printf("\n");
        
      return 0;
}
```

nams代码

```assembly
// start.asm
extern main
extern exit

_start:

		push eax
		push ecx
		call main
		; 为啥不需要从栈中清除调用main前入栈的两个元素？
		
		push eax
		; 所以，在执行应用程序的代码中，不需要exit。在这里执行exit了。
		call exit
```

execv修改进程的映像执行目标文件时，执行的第一个指令是`push eax`。

eax的值是do_exec中的`orig_stack`。

`esp`是`proc_table[src].regs.esp = PROC_IMAGE_SIZE_DEFAULT - PROC_ORIGIN_STACK;`。

`PROC_ORIGIN_STACK`是什么意思？我原以为，它是进程栈的大小。现在，我不这么认为了。

在execv中，替换进程的映象时，把`PROC_IMAGE_SIZE_DEFAULT - PROC_ORIGIN_STACK`设置成esp，执行指令时，eax、ecx依次入栈。

可见，进程的栈大小并不是`PROC_ORIGIN_STACK`。

进程的栈的大小其实在初始化进程时就已经设置了。

在操作系统中，栈分为好几种：

1. 内核栈
2. 进程栈
3. 有第三种吗？

> 又忘记了之前烂熟于心的知识。

我们在do_exec中修改的栈，其实，并不是进程栈。

我觉得，好像，在do_exec中修改的栈，是进程栈。

不理会这个问题了。不管它是不是进程栈，都不影响我理解start.asm。

入栈两个元素，然后调用main，orig_stack能满足要求。



又发现了一个非常奇怪的问题：

1. 进程的堆栈早在初始化时就已经确定了，在do_exec中修改的栈如果是进程的栈，二者的位置非常不一样啊。
2. 进程的堆栈在初始化时确定了，并不意味着，在do_exec中不能重新设置进程的栈。

在exec功能中，我有下面几条猜想：

1. 在execv中，`arg_stack`只是一个数组，并不是栈，只是被用来存储栈中的数据。
2. 在do_exec中，重新选择了一段内存空间作为栈空间。重新选择，当然需要重新放置数据，并且存储数据的内存地址。
3. orig_stack只是栈的一个中间地址，可以不是开始，也可以不是结尾。
4. 在入栈新元素前，栈顶是存储参数数据的内存的地址。
5. 入栈新元素后，完全能正确运行。不必管栈中存储了一些什么样的奇怪数据。

> 我在栈中存储的奇怪数据（参数、参数的内存地址、栈顶地址、参数个数）上浪费了很多时间。

1. 进程栈必须在进程的内存空间中吗？

### 编译

```shell
gcc -I ../include -c -fno-builtin -Wall -o pwd.o pwd.c
# -f的值不能设定为elf_i386。
nasm -I ../include -f elf -o start.o start.asm
ld -Ttext 0x1000 -o pwd pwd.o start.o my-crt.a
```

### 安装应用程序

```shell
dd if=inst.tar of=$(HD) seek=`echo "obase=10;ibase=16;(\`egrep -e '^ROOT_BASE' ../boot/include/load.inc | sed -e 's/.*0x//g'\`+\`egrep -e '#define[[:space:]]*INSTALL_START_SECT' ../include/sys/config.h | sed -e 's/.*0x//g'\`)*200" | bc` bs=1 count=`ls -l inst.tar | awk -F " " '{print $$5}'` conv=notrunc
```

这个脚本太复杂了。能分拆成单条脚本吗？

### 疑问

1. gcc的选项`-fno-stack-protector`是什么意思？
2. 又发现了一个非常奇怪的问题：
   1. 进程的堆栈早在初始化时就已经确定了，在do_exec中修改的栈如果是进程的栈，二者的位置非常不一样啊。
   2. 进程的堆栈在初始化时确定了，并不意味着，在do_exec中不能重新设置进程的栈。
3. 进程栈必须在进程的内存空间中吗？

> 所有的疑问，都应该记录下来。可以搁置，但不应该永远跳过。

4. 根设备是什么？根设备的初始扇区又是什么？

   1. ```shell
      # 计算结果是在硬盘的字节偏移量，十进制，安装应用程序的位置
      echo "obase=10;ibase=16;(4EFF+8000)*200" | bc
      ```

   2. 应用程序被写入硬盘的位置。

   3. 0x4EFF是怎么设置的？是从硬盘的第一个扇区开始还是从除开文件系统的第一个扇区开始？

   4. 0x8000是相对于0x4EFF的偏移量吗？

   5. 没有更多资料，盯着代码看也不能解决问题，我只能暂时跳过。在一个知识点上浪费太多时间，不可取。

## linux命令

### ar

文档：https://www.runoob.com/linux/linux-comm-ar.html

```shell
# ar -rv 打包成的文件名 要被打包的文件，不知道r的用途，v是显示打包过程
ar -rv one.bak ./*
# 显示打包成的文件中包含的文件名称
ar t one.bak
# 删除打包成的文件中后缀是.c的文件，d表示删除
ar d one.bak *.c
```

下面是具体应用

```shell
# /home/cg/yuyuan-os/osfs10/e/ar-test/lib
[root@localhost lib]# ar -rv one.bak ./*
ar: creating one.bak
a - ./close.c
a - ./exec.c
a - ./exit.c
a - ./fork.c
a - ./getpid.c
a - ./misc.c
a - ./open.c
a - ./orangescrt.a
a - ./printf.c
a - ./read.c
a - ./stat.c
a - ./string.asm
a - ./syscall.asm
a - ./syslog.c
a - ./unlink.c
a - ./vsprintf.c
a - ./wait.c
a - ./write.c
[root@localhost lib]# ls
close.c  exit.c  getpid.c  one.bak  orangescrt.a  read.c  string.asm   syslog.c  vsprintf.c  write.c
exec.c   fork.c  misc.c    open.c   printf.c      stat.c  syscall.asm  unlink.c  wait.c
[root@localhost lib]# ar t one.bak
close.c
exec.c
exit.c
fork.c
getpid.c
misc.c
open.c
orangescrt.a
printf.c
read.c
stat.c
string.asm
syscall.asm
syslog.c
unlink.c
vsprintf.c
wait.c
write.c
[root@localhost lib]# ar d one.bak *.c
[root@localhost lib]# ar t one.bak
orangescrt.a
string.asm
syscall.asm
[root@localhost lib]#
```

建立CRT（C运行时库）使用的命令是：`ar rcs`。不知其意。

### file

查看文件类型。

```c
[root@localhost lib]# file orangescrt.a
orangescrt.a: current ar archive
[root@localhost lib]# file string.asm
string.asm: UTF-8 Unicode text
```

### gcc

#### -fno-builtin

在定义函数的时候和C语言的内建函数重名了，导致编译的时候会报错：
`warning: conflicting types for built-in function ***`。
 平时一般都是想改改函数名不就得了，可突然冒出一个想法，能不能不改了，于是乎就发现了-fno-builtin这个选项。它的含义即不使用C语言的内建函数，用法如下：
`#gcc a.c -o a.out -fno-builtin`

#### -fno-builtin-function

  忽然又冒出了一个想法，要是我有的函数想用内建函数，有的不想用，那该怎么办了？ 于是乎我又发现了一个编译选项：
（其中function为要冲突的函数名），用法和上面的方法一致。

gcc的官方文档是：https://gcc.gnu.org/onlinedocs/gcc-8.5.0/gcc/Keyword-Index.html#Keyword-Index。

官方文档的内容太丰富了。我没能快速找到需要的知识。

想知道gcc的用法，问搜索引擎更快。

#### -c

编译为目标文件，不连接库。

-c意思就是Compile，产生一个叫helloworld.o的目标文件

#### -o 

指定输出文件的文件名,默认为a.out

#### -S

-S意思就是aSsemble，产生一个叫helloworld.s的汇编源文件。

***\*2.2\** 优化选项**

1) -O选项，基本优化：gcc -O helloworld.c

-O意思就是Optimize，产生一个经过优化的叫作a.out的可执行文件。也可以同时使用-o选项，以指定输出文件名。如：

gcc -O -o test helloworld.c

即会产生一个叫test的经过优化的可执行文件。

2) -O2选项，最大优化：gcc -O2 helloworld.c

产生一个经过最大优化的叫作a.out的可执行文件。

***\*2.3\** 调试选项**

***\*1) -g\**选项，产生供\**gdb\**调试用的可执行文件：\**gcc -g helloworld.c\****

产生一个叫作a.out的可执行文件，大小明显比只用-o选项编译汇编连接后的文件大。

2) -pg选项，产生供gprof剖析用的可执行文件：gcc -pg helloworld.c

产生一个叫作a.out的执行文件，大小明显比用-g选项后产生的文件还大。



#### -I 选项

指定包含的头文件的目录，这一点对于大型的代码组织来说是很有用的。

### -Wall

打印所有的警告信息。

#### 综合

```html
gcc常用选项
-c
编译生成目标文件（Relocatable），详见“main函数和启动例程”一节。

-Dmacro[=defn]
定义一个宏，详见“条件预处理指示”一节。

-E
只做预处理而不编译，cpp命令也可以达到同样的效果，详见“函数式宏定义”一节。

-g
在生成的目标文件中添加调试信息，所谓调试信息就是源代码和指令之间的对应关系，在gdb调试和objdump反汇编时要用到这些信息，详见“单步执行和跟踪函数调用”一节。

-Idir
dir是头文件所在的目录，详见“头文件”一节。

-Ldir
dir是库文件所在的目录，详见“静态库”一节。

-M和-MM
输出“.o文件: .c文件 .h文件”这种形式的Makefile规则，-MM的输出不包括系统头文件，详见“自动处理头文件的依赖关系”一节。

-o outfile
outfile输出文件的文件名，详见“main函数和启动例程”一节。

-O?
各种编译优化选项，详见“内联函数”一节。

-print-search-dirs
打印库文件的默认搜索路径，详见“静态库”一节。

-S
编译生成汇编代码，详见“main函数和启动例程”一节。

-v
打印详细的编译链接过程，详见“main函数和启动例程”一节。

-Wall
打印所有的警告信息，详见“第一个程序”一节。

-Wl,options
options是传递给链接器的选项，详见“共享库”一节。
```

### bc

```shell
# obase输出数据的进制，ibase输入数据的进制，bc计算前面的表达式例如16*2。
echo "obase=16;ibase=10;16*2;" | bc
20
```

### egrep

`egrep -e '^ROOT_BASE' ../boot/include/load.inc | sed -e 's/.*0x//g'`

从文件中找到需要的字符串，下面是应用举例：

```shell
# 关键词用单引号
[root@localhost e]# egrep -e '^ROOT_BASE' ./boot/include/load.inc | sed -e 's/.*0x//g'
4EFF
# 使用grep也可以
[root@localhost e]# grep -e '^ROOT_BASE' ./boot/include/load.inc | sed -e 's/.*0x//g'
4EFF
# 关键词用单引号和双引号混合
[root@localhost e]# grep -e '^ROOT_BASE' ./boot/include/load.inc | sed -e "s/.*0x//g"
4EFF
# 关键词全部用双引号
[root@localhost e]# grep -e "^ROOT_BASE" ./boot/include/load.inc | sed -e "s/.*0x//g"
4EFF
```



```shell
# #define[[:space:]]*INSTALL_START_SECT 中define和[[:space:]]之间不能有空格
[root@localhost e]# egrep -e '#define[[:space:]]*INSTALL_START_SECT'  ./include/sys/config.h | sed -e 's/.*0x//g'
8000
# 正则表达式中使用.*也可以
[root@localhost e]# egrep -e '#define.*INSTALL_START_SECT'  ./include/sys/config.h | sed -e 's/.*0x//g'
8000
```

### dd

```shell
# 把inst.tar写入80m.img；从80m.img的第27131392个字节开始写入；
# bs的意思，不清楚；
# count仅拷贝blocks个块，块大小等于ibs指定的字节数。
# conv=notrunc，不截短inst.tar
# bs=512 count=2，备份前2块总共为1024个字节的区域 。bs是单位，例如，512个字节。写入2个512个字节。
dd if=inst.tar of=../80m.img seek=27131392 bs=1 count=102400 conv=notrunc
```



Linux dd 命令用于读取、转换并输出数据。

dd 可从标准输入或文件中读取数据，根据指定的格式来转换数据，再输出到文件、设备或标准输出。

**参数说明:**

- if=文件名：输入文件名，默认为标准输入。即指定源文件。
- of=文件名：输出文件名，默认为标准输出。即指定目的文件。
- ibs=bytes：一次读入bytes个字节，即指定一个块大小为bytes个字节。
  obs=bytes：一次输出bytes个字节，即指定一个块大小为bytes个字节。
  bs=bytes：同时设置读入/输出的块大小为bytes个字节。
- cbs=bytes：一次转换bytes个字节，即指定转换缓冲区大小。
- skip=blocks：从输入文件开头跳过blocks个块后再开始复制。
- seek=blocks：从输出文件开头跳过blocks个块后再开始复制。
- count=blocks：仅拷贝blocks个块，块大小等于ibs指定的字节数。
- conv=<关键字>，关键字可以有以下11种：
  - conversion：用指定的参数转换文件。
  - ascii：转换ebcdic为ascii
  - ebcdic：转换ascii为ebcdic
  - ibm：转换ascii为alternate ebcdic
  - block：把每一行转换为长度为cbs，不足部分用空格填充
  - unblock：使每一行的长度都为cbs，不足部分用空格填充
  - lcase：把大写字符转换为小写字符
  - ucase：把小写字符转换为大写字符
  - swap：交换输入的每对字节
  - noerror：出错时不停止
  - notrunc：不截短输出文件
  - sync：将每个输入块填充到ibs个字节，不足部分用空（NUL）字符补齐。
- --help：显示帮助信息
- --version：显示版本信息

### 实例

在Linux 下制作启动盘，可使用如下命令：

```shell
dd if=boot.img of=/dev/fd0 bs=1440k 
```

将testfile文件中的所有英文字母转换为大写，然后转成为testfile_1文件，在命令提示符中使用如下命令：

```shell
dd if=testfile_2 of=testfile_1 conv=ucase 
```

其中testfile_2 的内容为：

```shell
$ cat testfile_2 #testfile_2的内容  
HELLO LINUX!  
Linux is a free unix-type opterating system.  
This is a linux testfile!  
Linux test 
```

转换完成后，testfile_1 的内容如下：

```shell
$ dd if=testfile_2 of=testfile_1 conv=ucase #使用dd 命令，大小写转换记录了0+1 的读入  
记录了0+1 的写出  
95字节（95 B）已复制，0.000131446 秒，723 KB/s  
cmd@hdd-desktop:~$ cat testfile_1 #查看转换后的testfile_1文件内容  
HELLO LINUX!  
LINUX IS A FREE UNIX-TYPE OPTERATING SYSTEM.  
THIS IS A LINUX TESTFILE!  
LINUX TEST #testfile_2中的所有字符都变成了大写字母 
```

由标准输入设备读入字符串，并将字符串转换成大写后，再输出到标准输出设备，使用的命令为：

```shell
dd conv=ucase 
```

输入以上命令后按回车键，输入字符串，再按回车键，按组合键Ctrl+D 退出，出现以下结果：

```shell
$ dd conv=ucase 
Hello Linux! #输入字符串后按回车键  
HELLO LINUX! #按组合键Ctrl+D退出，转换成大写结果  
记录了0+1 的读入  
记录了0+1 的写出  
13字节（13 B）已复制，12.1558 秒，0.0 KB/s 
```

### awk

```shell
[root@localhost e]# ls -l ./kernel.bin
-rwxr-xr-x. 1 root root 262952 Jun 13 00:30 ./kernel.bin
# 列的初始序号是1，所以，第1列是print $1。
[root@localhost e]# ls -l ./kernel.bin | awk '{print $1}'
-rwxr-xr-x.
[root@localhost e]# ls -l ./kernel.bin | awk '{print $5}'
262952
[root@localhost e]#
```



## 好资料

Linux C编程一站式学习 http://docs.linuxtone.org/ebooks/C&CPP/c/index.html

> 这也不算多好的资料。找本这个题材的书，都会包含这些知识。我的目标是写完操作系统，不可沉迷到这种资料中。
>
> 这个资料，能用来完善知识体系。

Linux学习资料

http://linux.vbird.org/linux_basic/

## v3

#### 八进制转十进制

我好差劲，这样的常规算法都需要理解一番！

把八进制字符串换算成十进制整数。

```c
#include <stdio.h>

int main(int argc, char * argv[])
{
  		char octal_str[10] = "18";
  		char *p = octal_str;
  		int decimal_num = 0;
  		while(*p){
        	decimal_num = decimal * 8 + (*p++ - '0');
      }
  
  		return 0;
}
```

#### untar

怎么写untar？

1. 读取一个扇区的数据D1。
2. 从D1中获取tar文件头H1。
3. 从H1中获取tar文件中的文件大小S1。
4. 读取((S1-1)/512 + 1)*512)个字节。
5. 把读取到的数据写入到新文件中。

> 和我平时做的CRUD有何不同？
>
> CRUD，基本不存在语法上的困惑，也很少有需求上的困惑，我似乎只需要充当一个需求翻译器。
>
> 解包tar文件呢？

1. 读取tar文件，最外层循环终止的条件是什么？

自己实现untar，不严谨的版本，耗费了很多时间。

##### 代码

```c
// file:/home/cg/yuyuan-os/osfs10/e/c-demo/untar.c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

struct tar_header{
        char name[100];
        char mode[8];
        char uid[8];
        char gid[8];
        char size[12];
        char mtime[12];
        char chksum[8];
        char typeflag;
        char linkname[100];
        char magic[6];
        char version[2];
        char uname[32];
        char gname[32];
        char devmajor[8];
        char devminor[8];
        char prefix[155];
        char padding[12];
};

int main(int argc, char * argv[])
{
        FILE *fd = fopen("cmd.tar", "r");
        // if(fd == -1){
        if(fd == NULL){
                printf("%s\n", "Open file failed");
                exit(1);
        }

        char buf[512*16];
        memset(buf, 0, 512*16);
        while(1){
          // int len = fread(buf, 512, 1, fd);
                int len = fread(buf, 512, 1, fd);
                if(len != 1)
                        break;

                // 上面的判断无法正确识别出无文件的情况。
                if(strlen(buf) == 0)
                        break;

                struct tar_header *header = (struct tar_header *)buf;
                char filename[30];
                memset(filename, 0, 30);
                strcpy(filename, header->name);

                int size = 0;
                char *p = header->size;
                char is_start = 1;
                while(*p){
                  			// 处理八进制字符串`"000023456"`，需要跳过第一个非零字符前面的所有的零。
                        if(is_start && (*p++ - '0') == 0){
                                // *p--;
                                continue;
                        }
                        if(is_start == 1){
                                *p--;
                                is_start = 0;
                        }
                        size = size * 8 + (*p++ - '0');
                }

                int bytes_left = size;
                int num = (bytes_left - 1)/512 + 1;

                //FILE *fdout = fopen(filename, "w");
                FILE *fdout = fopen(filename, "w+");
                char file_buf[512*16];
          			memset(file_buf, 0, 512*16);
                fread(file_buf, 512, num, fd);
                //char file_buf2[] = "hello";
                fwrite(file_buf, strlen(file_buf)+1, 1, fdout);
                fclose(fdout);
        }

        fclose(fd);
        return 0;
}
```



短短的一段代码，需要注意下面几点：

1. fopen、fread、fwrite等函数的用法。
2. 使用fwrite函数，需要执行fclose后，数据才写入到硬盘。
3. fread的返回值并不能用来判断是否已经读取完了所有数据，但可以通过检查读取到的数据的大小是否为0来判断。
4. 处理八进制字符串`"000023456"`，需要跳过第一个非零字符前面的所有的零。



##### linux命令

```shell
# 列出当前目录下的文件，并且排除test文件
[root@localhost untar]# ls | grep -v test
file.txt
fwrite-demo
fwrite-demo.c
untar.c
# 把当前目录下除test外的文件打包成cmd.tar文件
[root@localhost untar]# ls | grep -v test | xargs tar cvf cmd.tar
file.txt
fwrite-demo
fwrite-demo.c
untar.c
```

### 疑问

我实现的解包tar函数解包出来的文件末尾有一个奇怪的字符：` ^@`。

## 小结

### 2021-06-14 18:43

阅读ar、gcc等命令的用法。耗时1个小时3分钟。

搜索，找资料而已。

没有系统的书，问搜索引擎更快。官方文档的内容太多，不容易找到目标知识点。

### 2021-06-14 21:14

编写应用程序。耗1个小时32分。

在`start.asm`的调用main时传参那里浪费了非常多时间。

为啥慢？各种知识混合在一起，一时半会理解不清楚。

### 2021-06-15 00:04

阅读向硬盘中插入应用程序文件的shell代码。理解不了。无明确思路，瞎想。耗时43分。

### 2021-06-15 09:02

阅读把应用程序写入硬盘的脚本。耗时1个小时24分。

逐条学习几个linux命令：dd、bc、grep。思维不那么散漫，就能快些。

### 2021-06-15 12:19

首先说一句，我的操作系统笔记，写得不好。

1. 之前领悟过难点，没有记录，或者，不知道记录到哪里了。
2. 目录起不到索引的作用。
3. 正确和错误混合在一起。
4. 笔记，除了当时促进理解，更是以后查找的。一定要有正确的记录。

耗时2个小时23分。我做了什么？

1. 安装应用程序：读初始化文件系统代码、读untar代码，有很多细节，我不懂，例如MAKE_DEV。
2. 忘记了还有什么事情。记忆力真差！一定要写细粒度的小结。如此，我还能知道我做了些什么。

### 2021-06-15 13:21

理解“进制转十进制”的算法。耗时10 + 26分钟。

运行一次，就勉强理解了。总感觉，不接受这种算法。

又为

```c
#include <stdio.h>

int main(int argc, char * argv[])
{
  		char octal_str[10] = "18";
  		char *p = octal_str;
  		int decimal_num = 0;
  		while(*p){
        	decimal_num = decimal * 8 + (*p++ - '0');
      }
  
  		return 0;
}
```

中的`(*p++ - '0')`纠结了26分钟。

### 2021-06-15 15:37

看了一会文件系统。

我本来是想继续看完内存管理，可我一时不知道该看哪里，所以，我挑选一个文件从头到尾看。

我十多天前才看懂了文件系统，此刻，又忘记了。

先看完内存管理，再看文件系统。如果不妨碍我理解内存管理，暂时不用理会看不懂的文件系统。

先看懂，再写出来。

耗时20分钟。

### 2021-06-15 15:37

实现了解包tar文件的功能。用PHP写，会麻烦很多，我目前不知道怎么用PHP处理C中的struct。

耗时2个小时40分。

为啥慢？

1. 思路。所花时间不多，但却是感觉最困难、最耗时的。
2. C语言读写文件的函数，需要查文档，实验几次，才成功。
3. 调试几次才发现问题：
   1. 读取文件结束的判断方法，不是fread的返回值，而是读取到的数据的长度。
   2. 转换文件长度。tar文件头中的size。
   3. fwrite写函数，需要调用fclose后才生效。

