# 在内核中添加中断处理---外部中断

## 理论

### 无题

这是”在内核中添加中断处理“系列的第三个实验。

边写边忘。第一个实验，简单版本，做了些什么，我已经忘记了。及时复习，很有必要。

### 回忆

外部中断，是键盘、时钟等外部硬件产生的中断。

已经做过时钟中断的实验，核心是向8253写入counter0的初始值和设置counter0的模式。我已经忘记了细节。呜呜呜......

键盘中断，一大堆内容。也许是假象，起初，我也觉得内部中断有一大堆内容。

初始化8259A，向主从片的ICW1、ICW2、ICW3、ICW4写入配置，向主从片的OCW1写入数据来设置要放行的中断端口。

已经写好了8259A的初始化函数，用汇编写的。为了减小工作量，本次实验不用C重写8259A的初始化函数。

怎么开启键盘等外部中断？

1. 在初始化8259A时，放行键盘、时钟中断。
2. 在IDT中填充键盘等外部中断的中断项：
   1. 中断例程。
   2. 中断向量。
   3. 还有哪些？又忘记了。

语法是：

1. 在kernel.asm中写好中断例程，写成一个函数。
2. 在kernel.asm用global导出这些函数。
3. 在kernel_main.c中声明这些函数。
4. 在kernel_main.c创建一个函数，init_outer_interrupt，初始化外部中断的描述符并且把它们放入IDT。

复制粘贴工作较多。

难点是：

1. 约定俗成的外部中断：向量号、对应哪些设备
2. 怎么处理外部中断？
3. 第2点是重点和难点。

### 学习

#### v1

##### 建立中断例程

建立16个中断例程，表现形式是函数。16个函数分成两大类，

1. 第0~~~~第7个函数属于主片。内部是一个宏，宏调用`spurious_irq`函数。
2. 第8~~~第15个函数属于从片。内部是一个宏，宏调用`spurious_irq`函数。
3. 目前，两个宏仅仅是名称不同，内容相同。

16个函数在kernel.asm中，用`global`声明为全局函数。

`spurious_irq`在kernel_main.c中，用C语言写，打印中断向量。在kernel.asm中用`extern spurious_irq`导入。

##### 加入IDT

初始化一个中断描述符，需要：

1. 中断例程所在的代码段的选择子。
2. 中断例程偏移量。利用`typedef void (*int_handle) ()`实现。
3. 属性TYPE。
4. 属性Privilege。
5. 向量号。

只有向量号需要琢磨一会儿。

在建立中断例程时，传输给`spurious_irq`的是序号，不是向量号。一定要注意这点。起初，我把它当成向量号。

向量号出现在初始化中断描述符并把它加入IDT的时候。

在初始化8259A时，通过ICW2设置了主片和从片的初始向量号，假设，主片的初始向量号是：0x20，从片的向量号是：0x28。

那么，把中断描述符加入IDT时，

1. 挂接主片的描述符的向量号是：0x20~~~0x27。
2. 挂接从片的描述符的向量号是：0x28~~~0x30。

到这里，所有知识点都梳理完了。只剩一个问题：每个中断描述符对应什么外部中断？

若找不到其他渠道的资料，我只能默写于上神的代码咯。

`hwint`是什么意思？



```html
 IRQ#  Interrupt         Function

IRQ0     8      timer (55ms intervals, 18.2 per second)
IRQ1     9      keyboard service required
IRQ2     A      slave 8259 or EGA/VGA vertical retrace
IRQ8    70      real time clock  (AT,XT286,PS50+)
IRQ9    71      software redirected to IRQ2  (AT,XT286,PS50+)
IRQ10   72      reserved  (AT,XT286,PS50+)
IRQ11   73      reserved  (AT,XT286,PS50+)
IRQ12   74      mouse interrupt  (PS50+)
IRQ13   75      numeric coprocessor error  (AT,XT286,PS50+)
IRQ14   76      fixed disk controller (AT,XT286,PS50+)
IRQ15   77      reserved  (AT,XT286,PS50+)
IRQ3     B      COM2 or COM4 service required, (COM3-COM8 on MCA PS/2)
IRQ4     C      COM1 or COM3 service required
IRQ5     D      fixed disk or data request from LPT2
IRQ6     E      floppy disk service required
IRQ7     F      data request from LPT1 (unreliable on IBM mono)
```

这是从 https://docs.huihoo.com/help-pc/int-int_table.html 找到的，和于上神的代码中对应的IRQ不完全一致。我不知如何处理。

我决定，只初始化键盘、时钟、鼠标接口。

我只加上了键盘中断。第一次敲击键盘能触发键盘中断例程。

中断例程函数不能用`ret`结尾，宏必须用`hlt`结尾。理解不了。

打开时钟中断，需要复制粘贴一些代码，再弄吧。

## 写代码

代码在：/home/cg/os/pegasus-os/v18

## 调试

## 总结

## 操作系统源码

http://www.minix3.org/