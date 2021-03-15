# 键盘--扫描码--ASCII码--显示器上的字符

在上一篇，我讲了键盘操作会产生扫描码以及如何解析`Pause`键和`Print Screen`键的扫描码。

在这一篇，我会说清楚”键盘上的输入为什么会出现在显示器上“。

## 极简版

1. 我们敲击键盘，产生扫描码。
2. 操作系统获取扫描码，把扫描码解析成ASCII码。
3. 操作系统把ASCII码写入显存，显示器上就会打印出显存中的可打印字符。

## 解析扫描码

在上一篇，我建立了一个Make Code和按键ASCII码（部分按键如Enter是我自己设置的值）的映射数组`keyboardMap`。

按键不是`Pause`，也不是`Print Screen`，就进入下面的解析流程。

### 默认键值

1. 当接收到的扫描码只有一个，
2. 扫描码的类型是Make Code，
3. 这个Make Code的值是`V`，
4. 那么键值是`keyboardMap[V *3]`。

### 另一个键值

1. 接收到的第一个扫描码是Make Code，值是`V`。
2. 从`keyboardMap`中查询出被按下的键是`left_shift`或`right_shift`。
3. 设置`column`的值是1。
4. 接收到第二个扫描码是Make Code，值是`N`。
5. 从映射数组中查询这个扫描码对应的键值是：`keyboardMap[N * 3 + column]`。

注意，查询到的键值是`keyboardMap[N * 3 + column]`。这就是键盘上的`1`、`A`这类按键与`shift`键组合时的值，即默认值外的另一个值。

#### 提问

请想一想，点击`shift + A`键，读取扫描码的过程是什么样的？

### Enter

Enter键和Backspace键很容易解析。获取扫描码(Make Code)S，获取键值`keyboardMap[N * 3]`。

根据键值识别出是Enter键后就把光标设置到下一行。

识别出是Backspace就这样处理最新的两个字节：将高字节设置成`0Fh`，将低字节设置成空字符的ASCII码；另外，将光标的位置后退两个字符。

### Caps Lock等键

没有弄明白。

## 清屏

使用linux的命令行、显示器上打印满了字符，我们可以使用`clear`让命令行终端的字符全部消失。让屏幕上的字符全部消失，这就是清屏。

`clear`命令只是移动了光标的位置，并没有清除屏幕上的字符。按下方向键中的`Up`能定位到已经消失的字符。

我们的清屏，是让字符彻底从屏幕上消失，无论怎么按`Up`键都不会再看到字符。

清屏其实很简单，就是往写满字符的显存区域写入空字符。

## tty

### 什么是tty

我觉得`tty`是一个比较过时的功能，我几乎没用过。

想体验一番`tty`是什么的同学，可以在安装了linux系统的电脑或虚拟机上按下`shift + alt + f1`或`shift + alt + f2`键在不同的tty之间切换。下面是我在虚拟机上切换tty的效果，一个是图形界面，一个是纯命令行界面。

![image-20210312211841834](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210312211841834.png)



![image-20210312212013507](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/image-20210312212013507.png)

### VGA

我只讲述`VGA`模式下的tty。

如上图所示，不同的tty窗口展示的内容完全不相同，在tty0窗口敲击键盘，数据只会出现在tty0的显示器；切换到tty1后，显示器上不会出现刚刚敲击键盘的数据；反过来也是一样。

不同tty之间，是完全隔离的，只是公用一个键盘。

`VGA`是什么呢？我觉得弄清楚这个概念的方方面面没有更多的作用。对`VGA`，了解有限的下面这些有限的知识就够了。

#### 字符显示

在这种模式下，显示器一共能显示25行字符，每行80个字符。

每个字符占用2个字节，高字节是字符的颜色，低字节是字符的ASCII码。例如

```assembly
mov		ah,		0Ah
mov		al,		'A'
```

`mov ah, 0Ah`  中的 `0Ah` 的高4位和低4位分别是： `0000b` 和 `1010b`。它们分别设置字符的背景色和前景色。

`mov al, 'A'` 中的 `A` 是要打印的字符。

#### 寄存器

`VGA`模式下的显示器是一个硬件，提供了多组寄存器。

读写这些寄存器，能设置光标的位置、点击`Up`等方向键能滚屏。

读写这类寄存器，需先往一类寄存器中写入另一类寄存器的索引，即往另一类寄存器的第N个寄存器中写入数据；然后往第二类寄存器写入数据，不需要再指定具体是哪个寄存器。

### tty的实现

显示器上的内容是显存中的数据的映射。要让不同的tty窗口打印不同的内容，只需把显存分割成若干块，每个tty分配一块。

显示器一满屏所需空间是`80 * 25 * 2 = 4000`个字节。显存的内存地址是`0xB8000~~~0xBFFFF`，总计`0xBFFFF - 0xB8000 + 1 = 0x8000 = 32768`字节。如果实现3个tty，每个tty对应的显存大小大概是`32768 / 3 = 10922`个字节，能存储两”满屏“数据。

#### 伪代码

每个tty设计一个缓冲区C1，数据从键盘缓冲区C2到这个缓冲区，然后再从C1读取数据写入对应tty的显存区域。

tty对应的显存区域，也设计一个结构来存储，叫`Console`。

直接看伪代码吧。

````c
typedef		struct{
  	// tty使用的显存的开始位置
  	int		original_address;
  	// tty使用的显存的大小
  	int		limit;
  	// tty使用显存的当前位置
  	int		current_address;
  	// tty的光标位置
  	int		cursor_address;
}Console;

typedef		struct{
  		// 缓冲区下一个要处理的字符的
  		int		tail;
  		// 缓冲区下一个空闲位置
  		int		head;
  		// 缓冲区存储的数据的数量
  		int		count;
  		// 存储数据的数组
  		int		buff[256];
  
  		Console * console;
}TTY;
````

#### 全局视角下的流程

1. tty任务和其他用户进程（TestA等）一起初始化，并使用`restart`运行tty任务。
2. tty任务的流程如下：
   1. tty任务遍历所有的tty窗口。
   2. 如果被遍历到的tty窗口是当前tty，从C1缓冲区读取数据D。
   3. 把D转换成ASCII码，然后交给写显存模块打印到显示器。
3. 发生时钟中断，给tty任务建立快照，调度模块让用户进程上CPU运行。
4. 用户敲击键盘，8048监测到键盘操作，读取扫描码（第2套），传输给8042。
5. 8042把扫描码（第2套）转换成第1套，放入键盘缓冲区C2，通知8259A发生键盘中断。
   1. 在C2中的数据被取走前，8042不再接收新数据。
6. 键盘中断例程`save`当前进程，从C2读取数据，然后放入C1。
7. tty任务在某个时刻获得CPU控制权，执行第2步的流程。






