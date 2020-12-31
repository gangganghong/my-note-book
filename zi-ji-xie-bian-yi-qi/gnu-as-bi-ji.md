---
description: 学习一下gnu as汇编，不打算学得太深入，能看懂基本汇编，能写基本的汇编就中止。
---

# gnu as 笔记

## gnu as 笔记

Gnu as 又叫 AT&T 汇编，搜索时用关键词：AT&T。

上次玩汇编中断时记录的笔记实在太差了！

我不记得有没有记录笔记，只在word文档中找到一点记录。

### 快照

当前环境，永远都是，根据这个镜像：centos7\_i386\_gas.img，创建容器，并且挂载目录：

/Users/cg/data/code/study-compiler-java/study/gas:/home/mac

实际上，我根据这个镜像 `i386/centos` 创建容器。为啥？因为前面提到的镜像不存在。

究竟是怎么回事？等验证后，再更正这里的笔记。

我以前写的笔记，太不准确了。会不会没有保存文件，丢失了最新笔记？

按下面的命令，能随时新建一个容器并进入。

```text
docker run -ti -v /Users/cg/data/code/study-compiler-java/study/gas:/home/mac i386/centos /bin/bash
docker rename f0afb62cca23 gas-centos
docker exec -it gas-centos /bin/bash
```

#### 当前环境

docker run -ti -v /Users/cg/data/code/study-compiler-java/study/gas:/home/mac bdobyns/centos4.6\_i386 /bin/bash

Docker

时间消耗18分。

时间消耗1小23分：

1. 将centos8上的gas代码转移到mac上，
2. 将mac上的gas代码挂载到i386docker容器内。
3. 更新容器内的i386机器的编译器。
4. 混淆了新旧镜像创建的容器。

搭建环境太浪费时间了。弄好环境好后，一定要尽量保存、复用。

#### 环境

1. 使用VMware Funsion上面的centos8虚拟机。
2. 用命令行连接虚拟机：ssh root@172.16.64.132，密码是 123456。
3. 代码所在目录：/home/cg/code/gnuas
4. 主要资料：
   1. /Users/cg/Desktop/计算机基础/汇编语言程序设计.pdf
   2. 这本书配套代码在：/home/cg/code/test
   3. 这么重要的信息，我上次中断时为什么不做记录？害得我到处找。
5. 本笔记所在目录：/Users/cg/Documents/gitbook/my-note-book

### 学习清单

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

### 知识点

#### 定义数据元素

没有什么好方法，只是看书，没有完全按照上面的清单看书。

能看懂大部分内容。

#### for循环

不能完全跟上讲解逻辑，也看不懂部分代码，比如，变量赋值，看懂了基本模板。当然，大部分内容，我记不住。

#### 标准整数长度

记不住。

内存中，小尾数；寄存器中，大尾数。

没理解那个例子。书没讲清楚左和右哪边是低内存位置。

不过，既然是经典书，我相信它的内容是正确的，

![image-20201119075407062](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/my-note-book/image-20201119075407062.png)

左边是低内存位置，右边是高内存位置。

有点绕，没理解透彻。不管这些了，我只记住，内存中和寄存器中的存储是相反的，但处理器自动完成了转换。

#### 无符号整数

没啥难点，但也没记住。

8位、16位、32位、64位，分别由1个字节、2个字节、4个字节、8个字节组成。

#### 带符号整数

带符号数值:最高位表示符号。最高位是0，正数；最高位是1，负数。

反码：负值是对应正值的各位取反。

补码：正整数A，它的负值（补码）是A的反码+1。最优表示带符号整数的方法。

我觉得补码难点，因为它需要计算。其实就这点计算量，放在小学、初中、高中，那都不算什么。到了计算机里，我觉得它有点难掌握，第一，这需要记忆。第二，需要心算。

这么多汉字，是怎么记住的？真有必要记住这个补码计算方法，那一定是能够记住的。

不看笔记，复述一次，补码的计算方法是个：正整数A，求它的补码。先求出A的反码B，然后B+1就是A的补码。

![image-20201119082338542](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/my-note-book/image-20201119082338542.png)

想问题，要依赖已经认可的知识。上面这个说法，可以自己推导一下，8位的无符号数和带符号数，16位的、32位的、64位的，看看不是都是和这个结果保持一致。是的话，那就该承认这个说法是正确的。这就是数学老师教的“归纳法”。

不知道咋回事，我有时没有自发地用这些方法，就凭直觉去理解一些东西，希望冒出某个思路，可是却没凭空获得能认同这个说法的思路。

能推导出来，不表示不会忘记这个结论。

得出一个结论，然后记住结论，这是推导和记忆两件事情。

比如，我能用react做出东西，但是却回答不了那些网上的面试题，甚至也说不出我用过的那些知识点。因为，会用和不看资料复述出来，这是两回事。想复述，依靠的是记忆力。

~~带符号整数能表示的范围，这样理解，2位能表示的数字，分别是：~~

~~原数： 00、01、10、11~~

~~反码： 11、10、01、00~~

~~补码： 00、11、10、01~~

~~十进制： 0、1、2、3~~

~~负数： -2、-1、0、1~~

~~值： 2、1、0、1~~

~~二进制： 10、01、00、01~~

~~反码： 01、10、00、10~~

~~补码： 10、11、（不要对正数求反码，不符合此处的规则）~~

 ~~01 00~~

~~10、11是带符号的数，前者最大值是，~~

~~不带符号的数，全是正数，最大值是11，即十进制3。~~

~~带符号数，翻译成十进制数是：0、1、-1、-2。~~

~~3不是1的2倍。~~

~~最终正确的思路：~~

1. ~~2位二级制数，能出现的组合：00        01        10        11~~
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

#### BCD

![image-20201119110452290](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/my-note-book/image-20201119110452290.png)

#### 浮点数

科学计数法

浮点格式

二进制浮点格式

![image-20201119111509135](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119111509135.png)

这本书的信息密度太大，几乎每句话都需要逐字逐句看，而且还可能看不懂。

做了这么多年编程，不记得有遇到过这些知识点，先跳过了。

#### 泛读内容

字符串，移动、从内存读取、放入内存、比较字符串、搜索字符串，每个知识点都是那么陌生，几乎所有内容都需要从头开始慢慢理解、记忆。

我不打算细看、不打算弄懂每个细节。我说我看过这部分内容，可我面对别人，无法讲出任何东西，可能也无法回答别人的问题。

这样看来，我读了这部分书，好像没作用。不知道有没有作用，我期望有作用，我期望我在看汇编代码时有点头绪。

#### 函数

**定义函数处理**

```text
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

![image-20201119122419447](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119122419447.png)

这又没复杂的知识点和逻辑。如果是我，根据那种场景，我也会那样做。把返回地址放入堆栈，那么通过出栈获取函数参数，或间接寻址栈，都会导致返回地址从栈中丢失或不在栈顶。所以，把返回地址放入另外的寄存器、不受压栈出栈影响的寄存器当然是最方便的。

这又是我的一个思维缺陷。理解红线内容，根本无需知道哪个是栈顶。

**函数开头和结尾**

![image-20201119124301833](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119124301833.png)

1. Pushl %ebp会不会使栈顶元素是%ebp？
2. Movl %esp， %ebp 是不是把第1步中的ebp又赋值给了ebp？
   1. 有什么必要要这么做？
3. Movl %ebp, %esp；将栈顶指向ebp，ebp是返回地址，然后出栈将ebp赋值给ebp，这不是多余吗？既然ebp里面已经有需要的值？又何必使用这两步？

很耗费时间。理解不了。

**定义局部函数数据**

![image-20201119125915735](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119125915735.png)

理解不了这句话。是说，函数把任何数据压入堆栈，ESP寄存器不会发生变化？可是，ESP为啥不发生变化？

**清空堆栈**

![image-20201119133017633](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119133017633.png)

我不明白，为啥这样做就能把数据清除出堆栈。

理解不了，就跳过，在php中，echo "hello";能输出”hello"，而且还要在语句末尾加上逗号，我怎么没有觉得不理解。

不理解这个清除堆栈的原因，不妨碍我写汇编时这样写。

不能再这样看汇编书了，进度太慢。

#### 系统调用

Linux内核版本的格式是：linux-a.b.c，a是主版本号，b是次版本号，c是补丁号。

Linux内核具备三个基本功能：内存管理、文件系统（VFS）、设备管理。

系统调用在下面的文件定义

![image-20201119135945263](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119135945263.png)

查看系统调用：man 2 exit。2是man的第2页。

**常用系统调用**

![image-20201119140343585](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119140343585.png)

![image-20201119140408087](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119140408087.png)

对内存系统调用最陌生，未曾接触过其中一个。

#### 使用系统调用

exit系统调用：

```text
movl $1, %eax
int 0x80
```

**系统调用输入值**

给系统调用传参

![image-20201119141615432](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119141615432.png)

#### 复杂的系统调用返回值

类似场景，返回值不是一个整型、字符串等，而是一个class或struct。

**sysinfo系统调用**

![image-20201119142026788](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201119142026788.png)

```text
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

**函数例子**

```text
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
   nop    # NOP空指令什么都不做,但也被编译器用来作为分割符
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

```text
as -o sysinfo.o sysinfo.s
ld -o sysinfo sysinfo.o
```

需要使用`ld`。若使用`gcc -o sysinfo sysinfo.o`会报错，说`main`重复定义了。

运行结果是：

```text
[root@localhost chap12]# ./sysinfo
Hello,I am eric!
It is echo function!
```

**知识点**

1. 使用了全局变量
2. 系统调用write打印数据：系统调用 + 中断
3. 调用无参数汇编函数

**疑问**

1. 怎么编写有参数的汇编函数？

#### 调试

**使用gdb**

```text
as -gstabs -o sysinfo.o sysinfo.s
ld -o sysinfo sysinfo.o
# gdb
break _start
run
next
count    # 按照正常的方式运行代码
info registers # 显示所有寄存器的值
# 查看类似struct结构的数据
(gdb) x/d &uptime
0x6000c9:    25582
(gdb) x/d &procs
0x6000f1:    630
(gdb) x/d &load1
```

#### 小结

不打算详细看书了，通过用汇编编程和读汇编代码来学习汇编。

时间消耗3个多小时。

这本书信息密集，看得稍微快点，大部分内容就不能理解。在堆栈、函数传参、整型的补码那里耗费了最多时间。

从书中获得的最有用的是，知道了怎么编译汇编，怎么使用gdb调试汇编，怎么写函数，怎么通过系统中断来实现系统调用。

### 编程

#### 用无参数函数输出hello

```text
# 数据段
.section .data
msg:
        .ascii "Hello,World\n"
        len = . - msg
# 主函数
.section .text
.global _start
_start:
        nop    # 没有也不影响程序功能
        call echo    # 调用自定义函数 echo

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

        ret    # 自定义函数结束必须用这个指令
```

```text
[root@localhost as]# as -o hi.o hi.s
[root@localhost as]# ld -o hi hi.o
[root@localhost as]# ./hi
Hello,World
[root@localhost as]# file hi
hi: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, not stripped
```

```text
.section .data
msg:
        .int 9
output:
        .asciz  "ID is %d\n"
.section .text
.global _start
_start:
        nop
         #pushl $12
        #pushl $output
        call echo
        # addl $8, %esp

        movl $1, %eax
        int $0x80

.type echo, @function
echo:
        #pushl %ebp
        #movl %esp, %ebp
        #subl $8, %esp

        #movl 42, %edx
        #movl 4(%esp), %ecx
        #movl $1, %ebx
        #movl $4, %eax
        #int $0x80

        # int $0x80
        pushl $10
        pushl $output
        call printf
        addl $8, %esp

        #movl %ebp, %esp
        #popl %ebp
        ret
```

#### 有参数的函数

```text
.section .data
output:
        .ascii  "ID is %d\n"
        len = . - output
.section .text
.global _start
_start:
        nop
        pushl $len
        pushl $output
        call echo

        movl $1, %eax
        int $0x80

.type echo, @function
echo:
        pushl %ebp
        movl %esp, %ebp

        movl 8(%ebp), %ecx
        movl 12(%ebp), %edx
        movl $1, %ebx
        movl $4, %eax
        int $0x80

        movl %ebp, %esp
        popl %ebp
        ret
```

时间消耗：2小时+++。

这是一个重大突破。

**小结**

1. 书本提供的demo，用了其他我看不懂的知识，导致我无法根据那个例子了解写汇编函数的基本模板。
2. 从C代码反汇编出来的代码太冗长，是64位的，我无耐心去看，要看懂难度比较大。总之，我没花一点时间去研究它。
3. 根据书本和网络资料，我模仿写出了现在来看大部分正确、关键部分出错的函数。
   1. 这个函数能生成目标代码，但一直出错，比如段错误、打印出乱码。
   2. 面对这种情况，我如何处理？
      1. 在自定义函数中调用系统函数。
      2. 在自定义函数中手工调用系统函数
      3. 还有其他尝试。概况一下，无清晰思路，先改改看。和以前一样。
   3. 我以为断点调试能帮我，就转向使用gdb断点。
      1. 非官方i386 centos4上的gdb不能正常使用，在这里耗时很多。
      2. 我改用了centos官方镜像 i386 centos7 创建容器，gdb基本能正常使用了，但仍有信息说缺乏独立调试信息等。
         1. 搜资料，按照说明安装了一些东西，警告信息仍存在。我索性不管了。因为，能断点调试了。
      3. 断点调试，没帮到我，我不会用gdb。书上的命令，有的不能用。而且，我想看的的stack的中的数据，不知道怎么看。
      4. 基本上，gdb对我解决这个问题无用。
   4. docker操作，不熟悉，需看笔记。
      1. 根据新容器创建镜像。
      2. 根据新镜像创建容器。
      3. 挂载mac的文件夹。
      4. 删除镜像。
   5. 不打算用GDB解决问题后，我很幸运，搜索到了可以直接运行的gas函数代码。
      1. 两种向子函数传送数据的方式：一是寄存器，二是我已经写出了的错误代码。
      2. 运行了别人的带参数的函数代码，我立即明白了，从`stack`中取数据的代码`movl 8(%ebp), %eax`有问题。
         1. 第一个参数的位置，`8(%ebp)`。
         2. 第二个参数的位置，12\(%ebp\)。
         3. 第三个参数的位置，16\(%ebp\)。
         4. 记住两个重要的数字：起点数字是8，间隔是4。后者原因未知。前者的原因，函数的返回地址是 4，它的后面是参数。

#### a+b

```text
.section .data
output:
    .asciz  "Id %d\n"
.section .text
.global _start
_start:
    nop
    pushl $12
    pushl $20
    call add
    pushl %eax
    pushl $output
    call printf

    pushl $0
    call exit

.type   add, @function
add:
    pushl %ebp
    movl %esp, %ebp

    #addl 8(%ebp), 12(%ebp)
    movl 8(%ebp), %eax
    addl 12(%ebp), %eax

    movl %ebp, %esp
    popl %ebp
    ret
```

#### loop

正确的代码：

```text
.section .data
str:
    .asciz "The value is:%d\n"

.section .text
.global _start
_start:
    movl $3, %ecx
    movl $0, %eax
    jcxz done
loop1:
    pushl %ecx
    pushl %eax

    pushl %ecx
    pushl $str
    call printf

    popl %eax
    popl %ecx

    loop loop1
done:
    movl $1, %eax           #System call:   exit
    movl $0, %ebx
    int $0x80
```

在调用`printf`前，把ecx、eax的值存储到stack中，调用结束后，再取出来。

```text
.section .data
str:
    .asciz "The value is:%d\n"

.section .text
.global _start
_start:
    movl $30, %ecx
    movl $0, %eax
    jcxz done
loop1:
    movl %ecx, %edi
    pushl %ecx
    pushl $str
    call printf
    # call write start
    #movl $str, %ecx
    #movl $12, %edx
    #movl $1, %ebx
    #movl $4, %eax
    #int $0x80
    # call write end
    movl %edi, %ecx
    loop loop1
done:
    movl $1, %eax           #System call:   exit
    movl $0, %ebx
    int $0x80
```

和上段代码类型，不过，本段代码把ecx存储到了edi中。

非常奇怪的问题。

```text
.section .data
tip:
    .ascii "Start to run:\n"
str:
    .ascii "The value is:%d\n"
count:
    .int 0
max:
    .int 20

.section .text
.global _start
_start:
    movl $7, %ecx
    movl $0, %eax
    #pushl $tip
    #call printf
    #add $4, %esp
    jcxz done
loop1:
    pushl %eax
    pushl $str
    call printf
    add $8, %esp
    add %ecx, %eax
    loop loop1
done:
    pushl $0
    call exit
```

这段代码会进入死循环，如下：

```text
The value is:16
The value is:16
The value is:16
The value is:16
The value is:16
The value is:16
The value is:16
The value is:16^C
[root@30d2935db749 mac]#
```

我的预期，会依次输出7或6次，然后退出。

我已经找到了出现这个问题的原因。

Loop 指令等价于：ecx = ecx - 1;ecx是否等于0。在echo中，无论是直接调用printf还是自己手工用系统调用输出字符串，都改变了ecx的值。执行完echo后，ecx的值也许是8，也许是-2，总之就是不等于0。那么，ecx将永远不会等于0，于是，出现了死循环。

这又是一个极大的时间消耗点。大概消耗了三四个小时，让我心急如焚。

1. 出现异常后，我首先对比其他代码。其他代码和我的没有本质区别。除了loop的巨大灾难，没有对loop和ecx特别着墨。
2. 网络资料，只搜索一篇就是讲本问题的，但和我的问题，原因不同，那是因为环境不同导致的，如：DOS环境、16位、32位等。
3. 看了王爽的《汇编语言》loop部分，也没找到答案。看其他书，同样无果。总之，我看了不少网络资料，都没用。
4. 用GDB调试，第一次没有任何发现，反倒又陷入对GDB不能熟练使用的麻烦之中。实在没办法，又断点调试。单步调试，有确凿的证据表明，调用echo或write后，ecx突然变了。在手工调用write的代码中，我看到了ecx。我豁然开朗了。ecx被改变了，直接不是0，而且离0越来越远，永远不可能再是0。
5. 这个问题，不应该消耗这么多时间。用二分法，应该就能逐步定位问题，根本不用去看那么多无用的资料，浪费那么多时间。

```text
.section .data
tip:
    .ascii "Start to run:\n"
str:
    .ascii "The value is:%d\n"
count:
    .int 0
max:
    .int 20

.section .text
.global _start
_start:
    movl $7, %ecx
    movl $0, %eax
    #pushl $tip
    #call printf
    #add $4, %esp
    jcxz done
loop1:
    add %ecx, %eax
    loop loop1
done:
    pushl %eax
    pushl $str
    call printf
    add $8, %esp

    pushl $0
    call exit
```

这段代码却是正常的，等价于下列代码：

```c
for(int i = 0; i < 7; i++){
    sum += i;
}
```

#### 有int类型参数的函数

```text
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

```text
func_param_int.s:10: Error: invalid instruction suffix for `push'
func_param_int.s:19: Error: invalid instruction suffix for `push'
func_param_int.s:29: Error: invalid instruction suffix for `push'
func_param_int.s:30: Error: invalid instruction suffix for `push'
func_param_int.s:35: Error: invalid instruction suffix for `pop'
```

```text
(.text+0x14): undefined reference to `printf'
```

~~ld -static **-melf\_i386** -e nomain -o TinyHelloWorld TinyHelloWorld.o~~

~~ld -o func\_param\_int func\_param\_int.o -m elf\_i386~~

~~ld -static -melf\_i386 -e nomain -o func\_param\_int func\_param\_int.o~~

~~ld -static -melf\_i386 -o func\_param\_int func\_param\_int.o~~

### 小结

时间消耗点：

书上的代码是32位的，centos8是64位的。找了不少网络，资料，这些代码在64位机器上不能运行，目前已经发现的是push指令。

现在去学64位，难度太大。我决定安装一个i386版本的centos7试试。

VMWare Funsion 安装 centos 7 32位失败。

使用docker 镜像。

有用的资料：

i386版centos docker 镜像：[https://hub.docker.com/r/bdobyns/centos4.6\_i386/](https://hub.docker.com/r/bdobyns/centos4.6_i386/)

大学教授主页：[https://www3.nd.edu/~dthain/](https://www3.nd.edu/~dthain/)

Gas 64位汇编教程

[https://blog.csdn.net/pro\_technician/article/details/78173777?utm\_medium=distribute.pc\_relevant.none-task-blog-BlogCommendFromBaidu-1.control&depth\_1-utm\_source=distribute.pc\_relevant.none-task-blog-BlogCommendFromBaidu-1.control](https://blog.csdn.net/pro_technician/article/details/78173777?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.control)

Centos 下载：

[https://www.centos.org/download/](https://www.centos.org/download/)

Centos 7 32 位 官方docker 镜像：

[https://hub.docker.com/r/i386/centos](https://hub.docker.com/r/i386/centos)

```text
docker pull i386/centos
docker run -it i386/centos:latest
```

```text
[root@b626c50b3726 code]# ./test
hello,world!
[root@b626c50b3726 code]# gdb test
GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-120.el7
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i686-redhat-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /home/code/test...done.
(gdb) b main
Breakpoint 1 at 0x8048413: file test.c, line 6.
(gdb) b f
Breakpoint 2 at 0x8048425: file test.c, line 11.
(gdb) run
Starting program: /home/code/test
warning: Error disabling address space randomization: Operation not permitted
Cannot create process: Operation not permitted
During startup program exited with code 127.
(gdb)
```

**查看系统类型**

\[root@0b13ab8deb5a code\]\# file hello.o hello.o: ELF 32-bit LSB relocatable, Intel 80386, version 1 \(SYSV\), not stripped \[root@0b13ab8deb5a code\]\# file /sbin/init /sbin/init: ELF 32-bit LSB executable, Intel 80386, version 1 \(SYSV\), for GNU/Linux 2.2.5, dynamically linked \(uses shared libs\), stripped \[root@0b13ab8deb5a code\]\# getconf LONG\_BIT 32 \[root@0b13ab8deb5a code\]\# getconf LONG\_BIT 32 \[root@0b13ab8deb5a code\]\# ld -o hello -lc hello.o \[root@0b13ab8deb5a code\]\# ./hello bash: ./hello: /usr/lib/libc.so.1: bad ELF interpreter: No such file or directory \[root@0b13ab8deb5a code\]\#

\[root@0b13ab8deb5a code\]\# uname -a Linux 0b13ab8deb5a 4.9.125-linuxkit \#1 SMP Fri Sep 7 08:20:28 UTC 2018 x86\_64 x86\_64 x86\_64 GNU/Linux

\[root@0b13ab8deb5a code\]\# ld -o hello -lc hello.o \[root@0b13ab8deb5a code\]\# ./hello bash: ./hello: /usr/lib/libc.so.1: bad ELF interpreter: No such file or directory

\[root@0b13ab8deb5a code\]\# ./hello bash: ./hello: /lib/ld-linux.so.2.o: bad ELF interpreter: No such file or directory

报错：

ld: cannot find -lc

没有安装gcc导致的y

**正确的编译命令**

\[root@0b13ab8deb5a code\]\# ld -dynamic-linker /lib/ld-linux.so.2 -o hello -lc hello.o

终于可以了！这又是一次自己坑自己的悲剧！

看书只看一半，下一页写着，会出现问题，要怎么解决。我却没看。之前，在centos8上面，按那个方法，应该就没问题了。

而且，这个问题，前几天玩汇编的时候，我好像遇到过。记忆力真的下降了吗？以后要记好文档。

```text
$ sudo docker commit <当前运行的container id> <仓库名称>:<tag>
$ sudo docker save -o <仓库名称>-<tag>.img <仓库名称>:<tag>
示例如下:
$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
111111111111        222222222222        "/bin/bash"   5 minutes ago       Up 5 minutes                                       jello
$ sudo docker commit 111111111111 bash:1.0
$ sudo docker save -o bash-1.0.img bash:1.0
```

sudo docker commit 098a58dac17e bdobyns/centos4.6\_i386 $ sudo docker save -o centos4.6\_i386.img bdobyns/centos4.6\_i386

根据新容器保存镜像

```text
docker commit 30d2935db749 i386/centos:latest
docker save -o centos7_i386_gas.img i386/centos:latest
```

chugangdeMacBook-Pro:my-note-book cg$ docker commit b626c50b3726 i386/centos:latest sha256:24882c206bf37f037f50378fcaa71ad804eae6eb70c75f3131079100a22b0f96

```text
chugangdeMacBook-Pro:virtual cg$ docker rmi 99e6d3f782ad
Error response from daemon: conflict: unable to delete 99e6d3f782ad (must be forced) - image is referenced in multiple repositories
chugangdeMacBook-Pro:virtual cg$

解决方法：

docker rmi -f 99e6d3f782ad
```

```text
chugangdeMacBook-Pro:demo cg$ docker images | grep 'i386'
i386/centos                               latest              24882c206bf3        19 minutes ago      334MB
i386/centos                               <none>              fe70670fcbec        20 months ago       201MB
chugangdeMacBook-Pro:demo cg$ docker rmi fe70670fcbec
Error response from daemon: conflict: unable to delete fe70670fcbec (cannot be forced) - image has dependent child images
chugangdeMacBook-Pro:demo cg$
```

docker run -ti -v /Users/cg/data/code/study-compiler-java/study/gas:/home/mac i386/centos /bin/bash

进入被更新之后的镜像创建的容器

docker run -ti -v /Users/cg/data/code/study-compiler-java/study/gas:/home/mac bdobyns/centos4.6\_i386 /bin/bash

collect2: error: ld returned 1 exit status

在i386 centos7上，运行gdb出现下面的问题。搜索了很多网络资料，都解决不了。在centos64 8上面，根本没有这个问题。

在这个问题上耗费38分钟了。想办法跳过这个问题。

我写汇编遇到段错误，无法调试。想用gdb断点调试。

```text
[root@cbb2ff95679c mac]# gdb test
GNU gdb Red Hat Linux (6.3.0.0-1.153.el4_6.2rh)
Copyright 2004 Free Software Foundation, Inc.
GDB is free software, covered by the GNU General Public License, and you are
welcome to change it and/or distribute copies of it under certain conditions.
Type "show copying" to see the conditions.
There is absolutely no warranty for GDB.  Type "show warranty" for details.
This GDB was configured as "i386-redhat-linux-gnu"...Using host libthread_db library "/lib/tls/libthread_db.so.1".

(gdb) b main
Breakpoint 1 at 0x8048384: file test.c, line 4.
(gdb) b test
Breakpoint 2 at 0x80483b0: file test.c, line 11.
(gdb) run
Starting program: /home/mac/test
ID is:5
hello,world!

Program exited normally.
You can't do that without a process to debug.
(gdb)
```

```text
[root@acee2aea0a16 mac]# gcc -g -o test test.c
[root@acee2aea0a16 mac]# gdb test
GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-120.el7
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i686-redhat-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /home/mac/test...done.
(gdb) b main
Breakpoint 1 at 0x8048416: file test.c, line 4.
(gdb) run
Starting program: /home/mac/test
warning: Error disabling address space randomization: Operation not permitted
Cannot create process: Operation not permitted
During startup program exited with code 127.
```

解决：

使用docker的超级权限

docker run --privileged -ti -v /Users/cg/data/code/study-compiler-java/study/gas:/home/mac i386/centos /bin/bash

再次使用 gdb，又遇到问题：

```text
Missing separate debuginfos, use: debuginfo-install glibc-2.17-317.el7.i686
```

解决：

```text
先修改“/etc/yum.repos.d/CentOS-Debuginfo.repo”文件的 enable=1；
yum install nss-softokn-debuginfo --nogpgcheck
yum install yum-utils
debuginfo-install glibc-2.17-317.el7.i686
```

**搭建能断点调试的汇编开发环境成功**

能在i386 centos7上使用gdb调试了，耗费时间：2个小时。遇到的问题：

1. 在非官方centos i386镜像创建的容器内运行gdb
2. 我以为是使用gdb的命令不正确，搜索许久无果。
3. 换官方镜像创建容器，gdb仍不能运行。搜索知道，和docker启动的权限设置有关
4. 设置超级权限后，gdb提示缺少一些信息，安装。
5. 最后可以了。

![image-20201120124614228](/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201120124614228.png)

```text
(gdb) x/c $output
Value can't be converted to integer
```

gdb调试汇编

```text
(gdb) x/15i _start
=> 0x80481b0 <_start>:    push   $0xc
   0x80481b2 <_start+2>:    push   $0x804a014
   0x80481b7 <_start+7>:    call   0x80481c6 <echo>
   0x80481bc <_start+12>:    add    $0x8,%esp
   0x80481bf <_start+15>:    mov    $0x1,%eax
   0x80481c4 <_start+20>:    int    $0x80
   0x80481c6 <echo>:    push   %ebp
   0x80481c7 <echo+1>:    mov    %esp,%ebp
   0x80481c9 <echo+3>:    sub    $0x8,%esp
   0x80481cc <echo+6>:    mov    0x2a,%edx
   0x80481d2 <echo+12>:    mov    0x8(%esp),%ecx
   0x80481d6 <echo+16>:    mov    $0x1,%ebx
   0x80481db <echo+21>:    mov    $0x4,%eax
   0x80481e0 <echo+26>:    int    $0x80
   0x80481e2 <echo+28>:    push   $0xa
(gdb)
```

```text
(gdb) disassemble
Dump of assembler code for function _start:
   0x080481b0 <+0>:    push   $0xc
=> 0x080481b2 <+2>:    push   $0x804a014
   0x080481b7 <+7>:    call   0x80481c6 <echo>
   0x080481bc <+12>:    add    $0x8,%esp
   0x080481bf <+15>:    mov    $0x1,%eax
   0x080481c4 <+20>:    int    $0x80
End of assembler dump.
(gdb) p $output
$3 = void
(gdb) p $len
$4 = void
(gdb) n
12            call echo
(gdb) n

Breakpoint 1, echo () at int-param-func.s:20
20            pushl %ebp
(gdb) n
21            movl %esp, %ebp
(gdb) n
echo () at int-param-func.s:22
22            subl $8, %esp
(gdb) n
24            movl 42, %edx
(gdb) n

Program received signal SIGSEGV, Segmentation fault.
echo () at int-param-func.s:24
24            movl 42, %edx
(gdb) n

Program terminated with signal SIGSEGV, Segmentation fault.
The program no longer exists.

(gdb) x/42bc &output
0x804a004:    73 'I'    68 'D'    32 ' '    105 'i'    115 's'    32 ' '    37 '%'    100 'd'
0x804a00c:    10 '\n'    0 '\000'    0 '\000'    0 '\000'    1 '\001'    0 '\000'    0 '\000'    0 '\000'
0x804a014:    0 '\000'    0 '\000'    18 '\022'    0 '\000'    18 '\022'    0 '\000'    0 '\000'    0 '\000'
0x804a01c:    1 '\001'    0 '\000'    0 '\000'    0 '\000'    100 'd'    0 '\000'    0 '\000'    0 '\000'
0x804a024:    51 '3'    -127 '\201'    4 '\004'    8 '\b'    0 '\000'    0 '\000'    0 '\000'    0 '\000'
0x804a02c:    68 'D'    0 '\000'

(gdb) x/8bc &output
0x804a004:    73 'I'    68 'D'    32 ' '    105 'i'    115 's'    32 ' '    37 '%'    100 'd'
(gdb) x/4bc &output
0x804a004:    73 'I'    68 'D'    32 ' '    105 'i'
(gdb) x/12bc &output
0x804a004:    73 'I'    68 'D'    32 ' '    105 'i'    115 's'    32 ' '    37 '%'    100 'd'
0x804a00c:    10 '\n'    0 '\000'    0 '\000'    0 '\000'
(gdb) x/12bc &msg
0x804a000:    9 '\t'    0 '\000'    0 '\000'    0 '\000'    73 'I'    68 'D'    32 ' '    105 'i'
0x804a008:    115 's'    32 ' '    37 '%'    100 'd'
(gdb) x/d &msg
0x804a000:    9
(gdb) p/x $msg
$4 = Value can't be converted to integer.
(gdb) p/x $output
$5 = Value can't be converted to integer.
(gdb) p/x $ebp
$6 = 0x0
(gdb) x/i $ebp
   0x0:    Cannot access memory at address 0x0
(gdb) n
26            movl $output, %ecx
(gdb) n
27            movl $msg, %edx
(gdb) x/i ($ebp)
   0xffffd880:    add    %al,(%eax)
(gdb) x/i ($ebp+4)
   0xffffd884:    inc    %eax
(gdb)
(gdb) print output
$8 = 1763722313
(gdb) print msg
$9 = 9
(gdb) print/c output
$15 = 73 'I'
(gdb) print/c output+1
$16 = 74 'J'
(gdb) print/c output+8
$17 = 81 'Q'
(gdb) print/c output+9
$18 = 82 'R'
(gdb) print/c output+98
$19 = -85 '\253'
(gdb) print/c output+10
$20 = 83 'S'
(gdb) print $ebp
$22 = (void *) 0xffffd880
(gdb) p/x $pc
$23 = 0x804815c
(gdb) x/i $pc
=> 0x804815c <echo+21>:    mov    $0x4,%eax
(gdb) x/c output
0x69204449:    Cannot access memory at address 0x69204449
(gdb) x/3uh output
0x69204449:    Cannot access memory at address 0x69204449
(gdb) x/3uh msg
0x9:    Cannot access memory at address 0x9
(gdb) p    &0x804a004
Attempt to take address of value not located in memory.
(gdb) x/8c 0x804a004
0x804a004:    73 'I'    68 'D'    32 ' '    105 'i'    115 's'    32 ' '    37 '%'    100 'd'
(gdb)
(gdb) f
#0  _start () at int-param-func.s:11
11            pushl $msg
(gdb)
(gdb) info frame
Stack level 0, frame at 0x0:
 eip = 0x80481c4 in echo (printf-func.s:22); saved eip 0x80481c4
 Outermost frame: outermost
 source language asm.
 Arglist at unknown address.
 Locals at unknown address, Previous frame's sp in esp
(gdb) x/3x $esp
0xffffd884:    0x080481bd    0x0804a013    0x0000000c
(gdb) x/8c 0x080481bd
0x80481bd <_start+13>:    -72 '\270'    1 '\001'    0 '\000'    0 '\000'    0 '\000'    -51 '\315'    -128 '\200'    85 'U'
(gdb) x/8c 0x08048a013
0x8048a013:    Cannot access memory at address 0x8048a013
(gdb) x/8c 0x0804a013
0x804a013:    73 'I'    68 'D'    32 ' '    105 'i'    115 's'    32 ' '    37 '%'    100 'd'
(gdb)
```

[https://blog.csdn.net/xiaozi0221/article/details/90512542?utm\_medium=distribute.pc\_relevant\_t0.none-task-blog-BlogCommendFromBaidu-1.control&depth\_1-utm\_source=distribute.pc\_relevant\_t0.none-task-blog-BlogCommendFromBaidu-1.control](https://blog.csdn.net/xiaozi0221/article/details/90512542?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromBaidu-1.control&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromBaidu-1.control)

Gdb 使用

部分经过了验证

格式: x /nfu 

说明 x 是 examine 的缩写

n表示要显示的内存单元的个数

f表示显示方式, 可取如下值 x 按十六进制格式显示变量。 d 按十进制格式显示变量。 u 按十进制格式显示无符号整型。 o 按八进制格式显示变量。 t 按二进制格式显示变量。 a 按十六进制格式显示变量。 i 指令地址格式 c 按字符格式显示变量。 f 按浮点数格式显示变量。

u表示一个地址单元的长度 b表示单字节， h表示双字节， w表示四字节， g表示八字节

![image-20201120093019791](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201120093019791.png)

\_start+1，是什么意思？

```text
gdb打印表达式的值：print/f 表达式
f是输出的格式，x/d/u/o/t/a/c/f
表达式可以是当前程序的const常量，变量，函数等内容，但是GDB不能使用程序中所定义的宏
查看当前程序栈的内容: x/10x $sp-->打印stack的前10个元素
查看当前程序栈的信息: info frame----list general info about the frame
查看当前程序栈的参数: info args---lists arguments to the function
查看当前程序栈的局部变量: info locals---list variables stored in the frame
查看当前寄存器的值：info registers(不包括浮点寄存器) info all-registers(包括浮点寄存器)
查看当前栈帧中的异常处理器：info catch(exception handlers)
```

```text
[root@30d2935db749 mac]# ./c.sh add
filename:./c.sh
code:add
add.s: Assembler messages:
add.s: Warning: end of file not at end of a line; newline inserted
[root@30d2935db749 mac]#
```

```text
chugangdeMacBook-Pro:gas cg$ file hello
hello: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, not stripped
```

### 汇编知识点

#### 寄存器

**ESP和EBP**

ESP是栈顶指针，EBP是存取堆栈指针。

ESP就是一直指向栈顶的指针,而EBP只是存取某时刻的栈顶指针,以方便对栈的操作,如获取函数参数、局部变量等。

ESP：栈指针寄存器\(extended stack pointer\)，其内存放着一个指针，该指针永远指向系统栈最上面一个栈帧的栈顶。

EBP：基址指针寄存器\(extended base pointer\)，其内存放着一个指针，该指针永远指向系统栈最上面一个栈帧的底部。

![img](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/1597828-20190721145348151-751236658.png)

执行call add\_num时（调用函数），**ESP减4后将add\_num过程的返回地址压入堆栈，即当前指令指针EIP的值**（该值为主程序中call指令的下一条指令（不是push ebp）的地址）。_这一步是CPU完成的_。

很好的英语资料

[https://www.tenouk.com/Bufferoverflowc/Bufferoverflow2a.html](https://www.tenouk.com/Bufferoverflowc/Bufferoverflow2a.html)

![image-20201120183029116](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201120183029116.png)

![image-20201120183130351](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201120183130351.png)

![image-20201120183236726](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201120183236726.png)

#### 错误

```text
echo.s:11: Error: operand type mismatch for `mov'
```

## 从零开始搭建环境编写操作系统 AT&T GCC （四）绘制界面

[https://blog.csdn.net/cheng7606535/article/details/76653050](https://blog.csdn.net/cheng7606535/article/details/76653050)

## DOS系统功能调用表\(INT 21H\)

[https://blog.csdn.net/chinazeze/article/details/1735621](https://blog.csdn.net/chinazeze/article/details/1735621)

## GCC内嵌AT&T汇编语法

[https://blog.csdn.net/xsc\_c/article/details/17061407](https://blog.csdn.net/xsc_c/article/details/17061407)

### 9 内存引用

Intel语法的间接内存引用的格式为：

section:\[base+index\*scale+displacement\]

而在AT&T语法中对应的形式为：

section:displacement\(base,index,scale\)

其中，base和index是任意的32-bit base和index寄存器。scale可以取值1，2，4，8。如果不指定scale值，则默认值为1。section可以指定任意的段寄存器作为段前缀，默认的段寄存器在不同的情况下不一样。如果你在指令中指定了默认的段前缀，则编译器在目标代码中不会产生此段前缀代码。

下面是一些例子：

-4\(%ebp\)：base=%ebp，displacement=-4，section没有指定，由于base＝%ebp，所以默认的section=%ss，index,scale没有指定，则index为0。

foo\(,%eax,4\)：index=%eax，scale=4，displacement=foo。其它域没有指定。这里默认的section=%ds。

foo\(,1\)：这个表达式引用的是指针foo指向的地址所存放的值。注意这个表达式中没有base和index，并且只有一个逗号，这是一种异常语法，但却合法。

%gs:foo：这个表达式引用的是放置于%gs段里变量foo的值。

如果call和jump操作在操作数前指定前缀“_”，则表示是一个绝对地址调用/跳转，也就是说jmp/call指令指定的是一个绝对地址。如果没有指定"_"，则操作数是一个相对地址。

任何指令如果其操作数是一个内存操作，则指令必须指定它的操作尺寸\(byte,word,long），也就是说必须带有指令后缀\(b,w,l\)。

| **AT&T** **格式** | **Intel** **格式** |
| :--- | :--- |
| movl -4\(%ebp\), %eax | mov eax, \[ebp - 4\] |
| movl array\(, %eax, 4\), %eax | mov eax, \[eax\*4 + array\] |
| movw array\(%ebx, %eax, 4\), %cx | mov cx, \[ebx + 4\*eax + array\] |
| movb $4, %fs:\(%eax\) | mov fs:eax, 4 |

```text
#mov byte ptr es:[160 * 5 + 40 * 2],al
```

%es:\(160  _5 + 40_  2\)

#### 小结

昨天，我发现，在loop上使用write/print，将进入死循环。原因是write等破坏了循环中的ecx的值。

我的思路是，不使用write/print等打印字符，就像我七月份在看操作系统书时看到的`int 21H`打印字符那样。

奇怪的是，网络资料，都是用MANS、nams打印字符，极少有关于gas打印字符的（有，但都是调用printf、write）。

我找到了一个用gas写操作系统的博客系列，然而，太复杂了。我拷贝了代码，运行，段错误。

继续沿着这个思路搜索，也到书中找答案。可是，无收获。

有没有可能`int 21H`等同于`int 0X80`？只是二者是不同平台的？两者都叫系统调用嘛。

我的思路，很有可能是错误的。看stackoverflow，好像有个相关的问题，长长的一段英文，我几乎没耐心去看。看完了，仍然不是我需要的回答。它是纠正提问者将代码从32位改成64位。

后来，我又找到了直接回答我的问题的英文资料。

[https://cs.lmu.edu/~ray/notes/gasexamples/](https://cs.lmu.edu/~ray/notes/gasexamples/)

这份资料是64位模式下的代码。如果有能力，我更应该直接学64位。这个，学了就有用。可是，我没有相关资料，没有可供参考的。只能先学32位，尽管不会有作用。

在我解决了昨天遇到的循环打印字符串的问题后，我又面临一个问题：如何实现dos中的功能，在屏幕指定位置打印字符？

总共耗费时间：1小时55分。泛读资料，却没解决问题。可以定性为，浪费时间，低效。

#### 条件跳转

![image-20201121145110301](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201121145110301.png)

![image-20201121145333769](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201121145333769.png)

#### 循环指令

![image-20201121150446395](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201121150446395.png)

#### 遍历数组

```text
# signtest.s - An example of using the sign flag
.section .data
value:
   .int 21, 15, 34, 11, 6, 50, 32, 80, 10, 2
output:
   .asciz "The value is: %d\n"
.section .text
.globl _start
_start:
   movl $9, %edi
loop:
   pushl value(, %edi, 4)
   pushl $output
   call printf
   #addl $8, $esp # 不知有啥用，留着还报错
   decl %edi
   jns loop
   movl $1, %eax
   movl $0, %ebx
   int $0x80
```

`pushl value(, %edi, 4)`，很陌生的语法，不知道它的意思，也不会用。

#### 流程控制

**if**

![image-20201121151723849](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201121151723849.png)

**for**

![image-20201121152512871](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201121152512871.png)

for结构的第三个（一般是自增或自减），原来是和循环体一起执行的。这更新了我的认识。我一直将二者分为两个部分。

#### 处理字符串

使用gdb

```text
14       movsw
(gdb) x/s &output
0x804a018 <output>:    "T"
(gdb) s
15       movsl
(gdb) x/s &output
0x804a018 <output>:    "Thi"
(gdb) s
17       movl $1, %eax
(gdb) x/s &output
0x804a018 <output>:    "This is"
(gdb) s
18       movl $0, %ebx
(gdb) x/s &output
0x804a018 <output>:    "This is"
(gdb) s
19       int $0x80
```

看不懂书本。运行结果也和书本说明不一致。

### 定义数据元素

.data

.rodata 只读模式访问的数据

![image-20201121164947087](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201121164947087.png)

#### 定义静态符号

![image-20201121165136417](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201121165136417.png)

#### bss段

![image-20201121165503853](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201121165503853.png)

bss段的数据不必初始化，不会包含在程序中。

#### movx

![image-20201121165727613](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201121165727613.png)

**在内存和寄存器之间传递数据**

```text
# 从内存传到寄存器
movl value, %eax
# 从寄存器到内存
movl %eax, value
# 使用变址的内存地址
```

**变址的内存地址**

![image-20201121170421684](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201121170421684.png)

Offset\_address，是在base\_address的基础上偏移一定数量。

**使用间接寄存器寻址**

`movl %edx, -4(%edi)`

**数据交换指令**

![image-20201121172144772](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/image-20201121172144772.png)

#### 条件传送指令

如 `CMOV`

#### 小结

下午在干什么？

1. 看了if、for等实现方式。
2. 看了数据传输，还有其他一些语法。
3. 有些代码，不能运行通过。许多内容，只是浏览一次，能掌握的内容大概只有50%。像这样看下去，对我来说，没有用。等到一个月过去了、两个月过去了，我却没有拿得出来的东西，我又会觉得自己虚度了。笔记，肯定不能算是有用的东西。
4. 有用，是指能证明我的专业能力、能帮我得到更多优质面试机会、能帮我找到好工作。笔记，只不过让我知道多点知识罢了。
5. 下午看的这些内容，自然不是我的最终目的。我的最终目的是熟悉AT&T汇编，然后写完go编译器，并且为写操作系统打下汇编基础。
6. 对汇编，能看多快就看多快，能理解多少就理解多少，非基本知识（比如，if、for、写函数），遇到问题，实在解决不了，就先记下。不要懒得记。记下来，以后就可能回头看。不记，当时好像快点。可是，这种快，有什么效果呢？我要的是，能够懂得越来越多，原来不懂的东西，以后能有机会弄懂。
7. 不再看书，而是直接看汇编代码，看不懂再看书。

### 待整理的参考资料

汇编语言知多少\(四\): AT&T 汇编语法

[https://www.jianshu.com/p/0480e431f1d7](https://www.jianshu.com/p/0480e431f1d7)

### 64位 AT&T汇编的寄存器

1. 有16个常用的64位寄存器
2. %rax, %rbx, %rcx , %rdx, %rsi, %rdi, %rbp, %rsp \(和 8086汇编类似 \)
3. %r8, %r9, %r10, %r11, %r12, %r13, %r14, %r15
4. 寄存器的具体用途
5. **%rax** 作为函数返回值使用.
6. **%rsp** 指向栈顶.
7. %rdi,  %rsi,  %rdx,  %rcx,  %r8,  %r9,  %r10等寄存器用于存放函数参数.

![img](https://github.com/gangganghong/my-note-book/tree/e05c1a28d94eab12dcbaced04ebf95605897f21d/Users/cg/Documents/gitbook/my-note-book/zi-ji-xie-bian-yi-qi/1208639-9c29243af0edc5f8.png)

返回地址，是编译器自动入栈的。

