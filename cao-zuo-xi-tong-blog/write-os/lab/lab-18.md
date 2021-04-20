# 输入输出系统---TTY任务

## 代码位置

/home/cg/os/pegasus-os/v26

加一个附属小实验，测试一些零碎的东西。

/home/cg/os/pegasus-os/v26-1

## 回忆

> 记得的内容很少，尽快回忆完，重新学习。

### 显卡

和显示有关。

### 显存

gs指向的描述符描述的那段地址，就是显存。当然，显存不止那么大。那是80*25模式下的显存。

### tty

把所有显存划分成几个部分，每个tty使用一部分。屏幕显示当前tty。

## 学习

### v1

我实现的是简化的tty。

不同的tty使用同一个键盘、同一块显存、同一个屏幕。在屏幕上显示的内容不同，是因为显示的是同一块显存的不同区域。

计算机开机后，显示器的默认模式是80*25的文本模式。我的所有实验都在这个模式下进行。

这种模式下，显存的内存地址范围是：0xB8000~~0xBFFFF。我设置的gs指向的描述符，段界限是0xFFFF而不是0xBFFFF，是弄错了吗？

CRT Controlle Registers，是显示器（或叫视频）的众多寄存器中的一类。这类寄存器包含两类寄存器，一类是Data Registers，一类是Address Register。向Address Register写入地址。这个地址是Data Registers的索引，即，在这众多的Data Registers中选择那个寄存器写入数据。

![image-20210420073955907](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210420073955907.png)

CRT Controller Registers的读端口和写端口是同一个端口，奇怪。怎么用？

![image-20210420180952356](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210420180952356.png)



#### 滚屏

```c
// 0xC，选择的寄存器是 Start Address High Register
out_byte(0x3D4, 0xC);
out_byte(0x3D5, (80*15) >> 8);
// 0xD，选择的寄存器是 Start Address Low Register
out_byte(0x3D4, 0xD);
// 每行80个字符
out_byte(0x3D5, 80*15);
```

#### 光标跟随

```c
// 设置光标位置
// Cursor Location High Register
out_byte(0x3D4, 0x0E);
out_byte(0x3D5, (dis_pos/2) >> 8);
// Cursor Location Low Register
out_byte(0x3D4, 0xF);
out_byte(0x3D5, dis_pos/2);
```

> 写并调试滚屏和光标跟随，耗费2个多小时，时间主要消耗在滚屏。
>
> 为什么花这么多时间？
>
> 1. 理解于上神解析shift + UP的方法。代入具体数值才恍然大悟。最好不要心算。
> 2. 滚屏后不能打印新输入的字符。因为输入位置不在当前屏幕的可视区域。
> 3. 15行字符，滚屏后，不是恰好把字符隐藏了。
> 4. 先猜想，再验证，解决了问题。

> 先做零散实验，最后再独立写一次代码。

## 写代码

## 疑问

当前console才从键盘缓冲区（非8402缓冲）读数据，为什么？

缓冲区中的数据一定是当前console的吗？会不会是其他console的？

一直想不明白。

键盘中断发生的时候，也就是用户正在输入的时候，使用的console是当前console。

这个时候，tty也会获得运行机会。用户输入，是向当前console输入，放到缓冲区的数据是属于当前console的。因此，只有当前console才能读取数据，其他console读取数据是非法读取。

我不明白的问题：

缓冲区的数据还没有被当前console读取完，有剩余。这个时候，切换了console。这个console会读取缓冲区中的数据，包括不属于这个缓冲区的数据。

这种情况，怎么避免？

弄清楚这个问题，就基本没有问题了。

检测到用户按下切换控制台的按键时，先让当前console读取完缓冲区，然后再切换console。

检测缓冲区的counter是否为0，不为0，不切换，空转；

