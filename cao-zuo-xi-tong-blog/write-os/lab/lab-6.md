# 设置时钟中断频率

## 理论

### 回忆

在lab-5，我实现了时钟中断。外在效果是，屏幕上出现快速变换的字符，这就是“字符跳动”。

本实验的内容是，设置字符变换的时间间隔。

*忘记得太快了！我只模糊记得两个要点：一是要用到一个硬件；二是涉及到“赫兹”这个单位。*

#### 什么硬件

不记得这个硬件叫什么名字，暂且把它记作A。

这个硬件叫`8253`。

#### HZ

赫兹，是描述频率的，一秒钟内发生某种动作的次数。

A在N个时间单位内发生M次时钟中断。

能够设置M的值。N是固定的，通过改变M的值，能设置每次时钟中断之间的时间间隔。

本实验的任务是设置A的M值。

不是随意设置M的值，而是先确定时钟中断之间的时间间隔。

也就是说，本任务的两大重点：

1. 通过给出的时钟中断之间的时间间隔，求A的M值。
2. 用A能接受的方式设置A的M值。

再回忆。

8253有一个初始值，记作N；有一个计数器，记作C。

N是时钟中断的次数。

计数器的值从C递减至0，在这个时间内，发生N次时钟中断。

计数器每隔1秒钟减1。

C的值能够设定。设定一个值，意思是，在C秒内发生N次时钟中断。

《一个操作系统的实现》的讲解难以理解，不必拘泥它，8253的工作原理，就是我上面写的那样。

本实验的最大难点和重点，是怎么设置8253的计数器`counter0`的初始值，即C。

8253有多个计数器，其中一个是counter0。通过counter0设置8253的C。

怎么设置8253的C？

8253有三个计数器，分别是：couter0、couter1、couter2。

如果要把couter0的初始值设置为C，那么，依次把C的低8位、高8位通过0x40端口写入counter0。为什么不一次性写入？因为端口读写的单位是字节。

在向0x40端口写入数据前，必须先向0x43端口写入数据。这个数据的格式很复杂，我没有记住。由于不是应试考试，只需要弄清楚，书中对数据格式的描述和实际写入0x43的数据格式的对应关系即可。

这个问题，实际上是“大端法和小端法”问题。

又是这个问题。这一次，尽量弄清楚“大端法和小端法”吧。

~~什么叫大端法？高位数据存储在内存的高地址处，低位数据存储在内存的低地址处。~~

什么叫大端法？高位数据存储在内存的低地址处，低位数据存储在内存的高地址处。

例如，`0x12345678`存储在数组arr中，用大端法存储，结果是：

1. arr[3] = 78
2. arr[2] = 56
3. arr[1] = 34
4. arr[0] = 12

用小端法存储，结果是：

1. arr[3] = 12
2. arr[2] = 34
3. arr[1] = 56
4. arr[0] = 78

数组索引高，表示内存地址大。

简单说，小端法是“低地址放低位数据，高地址放高位数据”；大端法是“低地址放高位数据，高地址放低位数据”。

仍然需要通过记忆才能知道什么是大小端法，很容易忘记。

平时使用的十进制数，是小端法还是大端法？

这不是一个正确的问题，单独一个十进制数，不知道是大端法还是小端法。只有给出一个十进制和把它存储起来后的形式，才能判断是哪种。

bochs断点打印出来的全局描述符是大端法还是小端法？小端法。

xxd查看的数据是大端法还是小端法？不知道。不知道怎么构造测试数据来查看。

先这样吧。

再回到8253的0x43端口。

写入到0x43的数据用小端法还是大端法存储？不知道。这是最重要的知识点。

使用的是小端法。理由是什么？

向0x43写入的数据是`0x34`。`0x34`是这样存储起来的，`00 11 010 0`，左边是内存的高地址，右边是内存的低地址，符合“低地址存储低位数据”，所以，是小端法。

### 最终方案

“回忆”小结所写内容的后半部分基本是正确的，是重看书后写出来的。

详情见《一个操作系统的实现》6.5.2.1。

## 实践

代码在：`/home/cg/os/pegasus-os/v14`。

流程如下：

1. 要向8253的counter0写入的值是多少？下面是计算法方法：
   1. 欲设定时钟中断每10毫秒发生一次。
   2. 1秒钟发生100次时钟中断。
   3. 需要多少秒才能完成8253的全部时钟中断？1193180 / 100 = 11931.8秒。
   4. 1193就是counter0的值。
2. 把1193写入counter0。方法是：
   1. 向0x43端口写入0x34。
   2. 把1193的低8位写入0x40端口。
   3. 把1193的高8位写入0x40端口。
3. 上面的代码抽象成一个函数
   1. 函数名：Init8253
   2. 在开启中断前调用这个函数。

## 调试

基本没有问题。就是无法知道时钟频率设置是否生效了。

最大时间消耗点：测试时钟设置频率是否生效。挑了几个值，肉眼感知不到频率大小。

有方法能通过数字测试频率大小吗？不知道。

## 备注

不知道频率设置是否生效，这个实验不算真正完成。

搁置吧。