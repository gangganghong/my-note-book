---
description: 学习一下gnu as汇编，不打算学得太深入，能看懂基本汇编，能写基本的汇编就中止。
---

# gnu as 笔记



上次玩汇编中断时记录的笔记实在太差了！

我不记得有没有记录笔记，只在word文档中找到一点记录。

## 快照

### 当前环境

docker run -ti -v /Users/cg/data/code/study-compiler-java/study/gas:/home/mac bdobyns/centos4.6_i386 /bin/bash

Docker 

时间消耗18分。

### 环境

1. 使用VMware Funsion上面的centos8虚拟机。
2. 用命令行连接虚拟机：ssh root@172.16.64.132，密码是 123456。
3. 代码所在目录：/home/cg/code/gnuas
4. 主要资料：
   1. /Users/cg/Desktop/计算机基础/汇编语言程序设计.pdf
   2. 这本书配套代码在：/home/cg/code/test
   3. 这么重要的信息，我上次中断时为什么不做记录？害得我到处找。
5. 本笔记所在目录：/Users/cg/Documents/gitbook/my-note-book

## 学习清单

* [ ] 使用gdb调试
* [ ] 定义数据元素
* [ ] 传送数据元素
* [ ] 条件传送指令
* [ ] 交换数据
* [ ] 堆栈
* [ ] 指令指针
* [ ] 无条件分支
  * [ ] 跳转
  * [ ] 调用
  * [ ] 中断
* [ ] 条件分支
  * [ ] 条件跳转指令
  * [ ] 比较指令
  * [ ] 使用标志位
* [ ] 循环
  * [ ] 循环指令
  * [ ] 防止LOOP灾难
* [ ] if语句
* [ ] for循环
* [ ] 使用数字
  * [ ] 数字数据类型
  * [ ] 整型
  * [ ] SIMD整数
* [ ] 整数运算
* [ ] 移位指令
* [ ] 十进制运算
* [ ] 逻辑操作
* [ ] 定义函数
* [ ] 编写函数
* [ ] 访问函数
* [ ] 函数的放置
* [ ] 使用寄存器
* [ ] 常用系统调用
* [ ] 使用系统调用
* [ ] C库
* [ ] 打开和关闭文件
* [ ] 写入文件
* [ ] 读取文件
* [ ] 内存映射文件

凭感觉挑了上面这些知识点去熟悉。

相关例程都运行一次，理解。

最后，要写一个一百行左右的汇编程序，找四五个C程序翻译成汇编。

做到上面这些，就中止汇编的学习。

我学习汇编的目的，是为了写go编译器时把中间代码转为汇编，写操作系统时，能看懂汇编，使用汇编完成一些功能。

时间消耗：17分。纯粹是复制抄写。

## 知识点

### 定义数据元素

没有什么好方法，只是看书，没有完全按照上面的清单看书。

能看懂大部分内容。

### for循环

不能完全跟上讲解逻辑，也看不懂部分代码，比如，变量赋值，看懂了基本模板。当然，大部分内容，我记不住。

### 标准整数长度

记不住。

内存中，小尾数；寄存器中，大尾数。

没理解那个例子。书没讲清楚左和右哪边是低内存位置。

不过，既然是经典书，我相信它的内容是正确的，

![image-20201119075407062](/Users/cg/Documents/my-note-book/image-20201119075407062.png)



左边是低内存位置，右边是高内存位置。

有点绕，没理解透彻。不管这些了，我只记住，内存中和寄存器中的存储是相反的，但处理器自动完成了转换。

### 无符号整数

没啥难点，但也没记住。

8位、16位、32位、64位，分别由1个字节、2个字节、4个字节、8个字节组成。

### 带符号整数

带符号数值:最高位表示符号。最高位是0，正数；最高位是1，负数。

反码：负值是对应正值的各位取反。

补码：正整数A，它的负值（补码）是A的反码+1。最优表示带符号整数的方法。

我觉得补码难点，因为它需要计算。其实就这点计算量，放在小学、初中、高中，那都不算什么。到了计算机里，我觉得它有点难掌握，第一，这需要记忆。第二，需要心算。

这么多汉字，是怎能记住的？真有必要记住这个补码计算方法，那一定是能够记住的。

不看笔记，复述一次，补码的计算方法是个：正整数A，求它的补码。先求出A的反码B，然后B+1就是A的补码。

![image-20201119082338542](/Users/cg/Documents/my-note-book/image-20201119082338542.png)



想问题，要依赖已经认可的知识。上面这个说法，可以自己推导一下，8位的无符号数和带符号数，16位的、32位的、64位的，看看不是都是和这个结果保持一致。是的话，那就该承认这个说法是正确的。这就是数学老师教的“归纳法”。

不知道咋回事，我有时没有自发地用这些方法，就凭直觉去理解一些东西，希望冒出某个思路，可是却没凭空获得能认同这个说法的思路。

能推导出来，不表示不会忘记这个结论。

得出一个结论，然后记住结论，这是推导和记忆两件事情。

比如，我能用react做出东西，但是却回答不了那些网上的面试题，甚至也说不出我用过的那些知识点。因为，会用和不看资料复述出来，这是两回事。想复述，依靠的是记忆力。

~~带符号整数能表示的范围，这样理解，2位能表示的数字，分别是：~~

~~原数：		00、01、10、11~~

~~反码：		11、10、01、00~~

~~补码：		00、11、10、01~~

~~十进制：    0、1、2、3~~

~~负数：		-2、-1、0、1~~

~~值：			 2、1、0、1~~

~~二进制：    10、01、00、01~~

~~反码：		 01、10、00、10~~

~~补码：		 10、11、（不要对正数求反码，不符合此处的规则）~~

​					 ~~01    00~~

~~10、11是带符号的数，前者最大值是，~~

~~不带符号的数，全是正数，最大值是11，即十进制3。~~

~~带符号数，翻译成十进制数是：0、1、-1、-2。~~

~~3不是1的2倍。~~

~~最终正确的思路：~~

1. ~~2位二级制数，能出现的组合：00		01		10		11~~
2. ~~在这些组合中挑选出补码，剩余的是带符号的正整数。~~
3. ~~无符号整数3，求-3的二进制补码形式~~
   1. ~~二进制：11~~
   2. ~~取反：00~~
   3. ~~补码：01~~
   4. ~~-3用补码二进制表示是：01 吗？意外！~~
      1. ~~11 + 01 = 00，结果确实是十进制0。~~



上面的推导（或者叫理解），全部都是错误，或者说，可能不正确。

这个问题，这样去掌握。

1. 负数补码的计算方法，确实是其绝对值的二进制的反码+1。
2. 在《深入理解计算机系统》上给出了一个根据补码计算它表示的十进制的原理，直接用那个原理去计算就行了。
   1. 这个原理，计算的是所有所有有符号整数，包括正整数。
   2. 正整数也有补码，它的补码就是它自身。
3. 为啥这个原理是正确的，怎么得出这个原理的，书上没讲，我也想不出来。
4. 不知道第3点，对我知道2位二进制数能表示的最大数和最小数，毫无影响；知道了第3点，对我似乎也没啥帮助，时间久了，我可能还会忘记。所以，不要再在这种问题上浪费时间。
5. 2位二进制补码的所有情况：00      01    10    11
   1. 最小数是：10
   2. 最大数是：01
   3. 没有为什么，知道了也没啥用，不知道也没啥坏处。我要做的，只是记住这个规则。然后用原理给出的那个公式，计算出这两个补码表示的十进制分别是：-2、1。
6. 2位无符号二进制的所有情况：00    01    10    11
   1. 最小数是：00
   2. 最大数是：11
   3. 转为十进制，分别是：0、3。
7. 上图中说，”带符号整数的最大值是无符号值的一半“，显然是不对的。因为，1 不是 3 的 1半。



时间消耗2个小时左右，搞得我思维互相矛盾，无法自圆其说，很沮丧，连这点东西都看弄不明白，还写什么编译器和操作系统啊。

对陌生知识，我的心算能力很弱，实在不行，借住纸和笔计算吧。

还有，有些东西，可能不需要知道为啥是这样。如果理解不了，不理解又不妨碍我去实现主要目的的话，那就跳过吧。我不是跳过许多了吗？为啥非要在这里死磕？



### BCD

![image-20201119110452290](/Users/cg/Documents/my-note-book/image-20201119110452290.png)

### 浮点数

科学计数法

浮点格式

二进制浮点格式

![image-20201119111509135](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119111509135.png)

这本书的信息密度太大，几乎每句话都需要逐字逐句看，而且还可能看不懂。

做了这么多年编程，不记得有遇到过这些知识点，先跳过了。

### 泛读内容

字符串，移动、从内存读取、放入内存、比较字符串、搜索字符串，每个知识点都是那么陌生，几乎所有内容都需要从头开始慢慢理解、记忆。

我不打算细看、不打算弄懂每个细节。我说我看过这部分内容，可我面对别人，无法讲出任何东西，可能也无法回答别人的问题。

这样看来，我读了这部分书，好像没作用。不知道有没有作用，我期望有作用，我期望我在看汇编代码时有点头绪。

### 函数

#### 定义函数处理

```asm
.type func1,    @function
func1:
	<some code>
	RET
```

函数定义的结束符必须是RET。执行RET时，程序控制从函数内的指令返回到主程序的位置，这个位置是调用函数指令后面的下一条指令，即CALL指令后面的指令。

函数的知识点，比浮点数这些还好懂一些，但看一次仍然理解不了。

理解不了函数调用、函数返回的压栈、出栈过程。

假设内存地址是0----1023，栈顶是0还是1023？

按书中的说法，栈顶地址是最低的，因此，我认为0是栈顶。压栈，将栈指针减8，然后再将值存储到当前位置。那么，0-8是什么内存？

这个说法不行。

1023是栈顶，这样就说得通了。

在x86-64中，栈向低地址方向增长。单独看这句，很好理解。结合前面的”栈顶元素的地址是栈中所有元素地址中最低的“，就不好理解了。

压栈、出栈很好理解。哪个是栈顶，我纠结了很久。

对堆栈栈顶的理解，是由下图中红线引发的。

![image-20201119122419447](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119122419447.png)

这又没复杂的知识点和逻辑。如果是我，根据那种场景，我也会那样做。把返回地址放入堆栈，那么通过出栈获取函数参数，或间接寻址栈，都会导致返回地址从栈中丢失或不在栈顶。所以，把返回地址放入另外的寄存器、不受压栈出栈影响的寄存器当然是最方便的。

这又是我的一个思维缺陷。理解红线内容，根本无需知道哪个是栈顶。

#### 函数开头和结尾

![image-20201119124301833](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119124301833.png)

1. Pushl %ebp会不会使栈顶元素是%ebp？
2. Movl %esp， %ebp 是不是把第1步中的ebp又赋值给了ebp？
   1. 有什么必要要这么做？
3. Movl %ebp, %esp；将栈顶指向ebp，ebp是返回地址，然后出栈将ebp赋值给ebp，这不是多余吗？既然ebp里面已经有需要的值？又何必使用这两步？

很耗费时间。理解不了。

#### 定义局部函数数据

![image-20201119125915735](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119125915735.png)

理解不了这句话。是说，函数把任何数据压入堆栈，ESP寄存器不会发生变化？可是，ESP为啥不发生变化？

#### 清空堆栈

![image-20201119133017633](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119133017633.png)

我不明白，为啥这样做就能把数据清除出堆栈。

理解不了，就跳过，在php中，echo "hello";能输出”hello"，而且还要在语句末尾加上逗号，我怎么没有觉得不理解。

不理解这个清除堆栈的原因，不妨碍我写汇编时这样写。

不能再这样看汇编书了，进度太慢。

### 系统调用

Linux内核版本的格式是：linux-a.b.c，a是主版本号，b是次版本号，c是补丁号。

Linux内核具备三个基本功能：内存管理、文件系统（VFS）、设备管理。

系统调用在下面的文件定义

![image-20201119135945263](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119135945263.png)

查看系统调用：man 2 exit。2是man的第2页。

#### 常用系统调用

![image-20201119140343585](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119140343585.png)

![image-20201119140408087](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119140408087.png)



对内存系统调用最陌生，未曾接触过其中一个。

### 使用系统调用

exit系统调用：

```assembly
movl $1, %eax
int 0x80
```

#### 系统调用输入值

给系统调用传参

![image-20201119141615432](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119141615432.png)

### 复杂的系统调用返回值

类似场景，返回值不是一个整型、字符串等，而是一个class或struct。

#### sysinfo系统调用

![image-20201119142026788](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119142026788.png)

```assembly
_start:
   nop
   movl $result, %ebx
   movl $116, %eax
   int $0x80

   movl $0, %ebx
   movl $1, %eax
   int $0x80
```

`movl $1, %eax` 是什么意思？依照书的意思，是把`movl $116, %eax`系统调用的结果赋值给result。不理解。

#### 函数例子

```assembly
# sysinfo.s - Retrieving system information via kernel system calls
.section .data
msg:
        .ascii "Hello,I am eric!\n"
        len = . - msg
tip:
        .ascii "It is echo function!\n"
        tip_len = . - tip
# 定义类似struct的复杂结构
result:
uptime:
   .int 0
load1:
   .int 0
load5:
   .int 0
load15:
   .int 0
totalram:
   .int 0
freeram:
   .int 0
sharedram:
   .int 0
bufferram:
   .int 0
totalswap:
   .int 0
freeswap:
	 .int 0
procs:
   .byte 0x00, 0x00
totalhigh:
   .int 0
memunit:
   .int 0
.section .bss
        .lcomm words, 128
.section .text
.globl _start
_start:
   nop	# NOP空指令什么都不做,但也被编译器用来作为分割符
   movl $result, %ebx
   # sysinfo的数字码是116，32位
   movl $116, %eax
   # 0x80，系统中断，执行系统调用
   int $0x80
	
	 # 使用 系统调用write + 系统中断输出数据到屏幕
   movl         $len, %edx
   movl         $msg, %ecx
   # write 的第一个参数，文件句柄
   movl         $1, %ebx
   movl         $4, %eax
   int $0x80

   call echo

   movl $0, %ebx
   movl $1, %eax
   int $0x80

.type echo, @function
echo:
        movl    $tip_len, %edx
        movl    $tip, %ecx
        movl    $1, %ebx
        movl    $4, %eax
        int $0x80
```

编译命令：

```shell
as -o sysinfo.o sysinfo.s
ld -o sysinfo sysinfo.o
```

需要使用`ld`。若使用`gcc -o sysinfo sysinfo.o`会报错，说`main`重复定义了。

运行结果是：

```shell
[root@localhost chap12]# ./sysinfo
Hello,I am eric!
It is echo function!
```

##### 知识点

1. 使用了全局变量
2. 系统调用write打印数据：系统调用 + 中断
3. 调用无参数汇编函数

##### 疑问

1. 怎么编写有参数的汇编函数？



### 调试

#### 使用gdb

```shell
as -gstabs -o sysinfo.o sysinfo.s
ld -o sysinfo sysinfo.o
# gdb
break _start
run
next
count	# 按照正常的方式运行代码
info registers # 显示所有寄存器的值
# 查看类似struct结构的数据
(gdb) x/d &uptime
0x6000c9:	25582
(gdb) x/d &procs
0x6000f1:	630
(gdb) x/d &load1
```

### 小结

不打算详细看书了，通过用汇编编程和读汇编代码来学习汇编。

时间消耗3个多小时。

这本书信息密集，看得稍微快点，大部分内容就不能理解。在堆栈、函数传参、整型的补码那里耗费了最多时间。

从书中获得的最有用的是，知道了怎么编译汇编，怎么使用gdb调试汇编，怎么写函数，怎么通过系统中断来实现系统调用。

## 编程

### 用无参数函数输出hello

```assembly
# 数据段
.section .data
msg:
        .ascii "Hello,World\n"
        len = . - msg
# 主函数
.section .text
.global _start
_start:
        nop	# 没有也不影响程序功能
        call echo	# 调用自定义函数 echo
				
				# 利用0x80中断 实现系统调用(exit的code是1，32位机器)
        movl $1, %eax
        int $0x80

# 自定义函数
.type echo, @function
echo:
        movl $len, %edx
        movl $msg, %ecx
        movl $1, %ebx
        # 系统中断 + write系统调用
        # 上面三个是write的三个参数
        movl $4, %eax
        int $0x80
        
        ret	# 自定义函数结束必须用这个指令
```

```shell
[root@localhost as]# as -o hi.o hi.s
[root@localhost as]# ld -o hi hi.o
[root@localhost as]# ./hi
Hello,World
[root@localhost as]# file hi
hi: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, not stripped
```



### 有int类型参数的函数

```assembly
.section .data
msg:
        .int 9
output:
        .asciz  "ID is %d\n"
.section .text
.global _start
_start:
        # nop
        pushl $10
        call echo
        addl $4, %esp

        movl $1, %eax
        int $0x80

.type echo, @function
echo:
        pushl %ebp
        movl %esp, %ebp
        subl $4, %esp

        # movl 4, %edx
        # movl 8(%esp), %ecx
        # movl $1, %ebx
        # movl $4, %eax

        # int $0x80
        pushl $10
        pushl $output
        call printf
        addl $8, %esp

        movl %ebp, %esp
        popl %ebp
        ret
```

报错：

```shell
func_param_int.s:10: Error: invalid instruction suffix for `push'
func_param_int.s:19: Error: invalid instruction suffix for `push'
func_param_int.s:29: Error: invalid instruction suffix for `push'
func_param_int.s:30: Error: invalid instruction suffix for `push'
func_param_int.s:35: Error: invalid instruction suffix for `pop'
```

```
(.text+0x14): undefined reference to `printf'
```

~~ld -static **-melf_i386** -e nomain -o TinyHelloWorld TinyHelloWorld.o~~

~~ld -o func_param_int func_param_int.o -m elf_i386~~

~~ld -static -melf_i386 -e nomain -o func_param_int func_param_int.o~~

~~ld -static -melf_i386  -o func_param_int func_param_int.o~~

## 小结



时间消耗点：

书上的代码是32位的，centos8是64位的。找了不少网络，资料，这些代码在64位机器上不能运行，目前已经发现的是push指令。

现在去学64位，难度太大。我决定安装一个i386版本的centos7试试。

VMWare Funsion 安装 centos 7 32位失败。

使用docker 镜像。

有用的资料：

i386版centos docker 镜像：https://hub.docker.com/r/bdobyns/centos4.6_i386/

大学教授主页：https://www3.nd.edu/~dthain/

Gas 64位汇编教程

https://blog.csdn.net/pro_technician/article/details/78173777?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.control

Centos 下载：

https://www.centos.org/download/



Centos 7 32 位 官方docker 镜像：

https://hub.docker.com/r/i386/centos



#### 查看系统类型

[root@0b13ab8deb5a code]# file hello.o
hello.o: ELF 32-bit LSB relocatable, Intel 80386, version 1 (SYSV), not stripped
[root@0b13ab8deb5a code]# file /sbin/init
/sbin/init: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), for GNU/Linux 2.2.5, dynamically linked (uses shared libs), stripped
[root@0b13ab8deb5a code]# getconf LONG_BIT
32
[root@0b13ab8deb5a code]# getconf LONG_BIT
32
[root@0b13ab8deb5a code]# ld -o hello -lc hello.o
[root@0b13ab8deb5a code]# ./hello
bash: ./hello: /usr/lib/libc.so.1: bad ELF interpreter: No such file or directory
[root@0b13ab8deb5a code]#

[root@0b13ab8deb5a code]# uname -a
Linux 0b13ab8deb5a 4.9.125-linuxkit #1 SMP Fri Sep 7 08:20:28 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux

[root@0b13ab8deb5a code]# ld -o hello -lc hello.o
[root@0b13ab8deb5a code]# ./hello
bash: ./hello: /usr/lib/libc.so.1: bad ELF interpreter: No such file or directory

[root@0b13ab8deb5a code]# ./hello
bash: ./hello: /lib/ld-linux.so.2.o: bad ELF interpreter: No such file or directory

报错：

ld: cannot find -lc

没有安装gcc导致的y

#### 正确的编译命令

[root@0b13ab8deb5a code]# ld -dynamic-linker /lib/ld-linux.so.2.o -o hello -lc hello.o

终于可以了！这又是一次自己坑自己的悲剧！

看书只看一半，下一页写着，会出现问题，要怎么解决。我却没看。之前，在centos8上面，按那个方法，应该就没问题了。

而且，这个问题，前几天玩汇编的时候，我好像遇到过。记忆力真的下降了吗？以后要记好文档。









