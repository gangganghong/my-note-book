# 输入/输出系统---显示器

## tty

使用`Alt + Fn`切换到不同的tty。

tty对用户来说，外在表现是一个`Console`，也就是一个终端。

tty不止一个，也许有三个，例如，tty1、tty2、tty3。

敲击同一个键盘（比如我现在写作用的电脑只有一个键盘），只会在当前tty输出字符，就像我们看到的，只在当前显示器显示字符。

这是怎么做到的？

把显存分为三块，每个tty对应一块。敲击键盘，只会把数据写入当前tty对应的显存。

tty共用一个键盘，共用一块显存，独立使用这块显存的不同部分。

## 显示器

### 80*25

目前，我们把显示器设置为`80*25`文本模式。

在这个模式下，显示器每行显示80个字符，最大能显示25行。

每个字符占用2个字节，一个字节是字符的ASCII码，另一个字节是字符的颜色模式（前景色和背景色）。

使用完整个显示器的当前空间需要`80*25*2 = 4000`个字节。

我们的显存的范围是`0xB8000~~~0xBFFFF`，一共`0x07FFF + 1 = 0x08000 =32KB `。

### 屏幕上的字符的字节构成

![image-20210312110416395](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/note/image-20210312110416395.png)

低字节是字符，高字节是字符的颜色模式。

高字节又等分成两组（每组4个bit，分别表示前景色和背景色），每3个bit表示一种颜色（RGB模式），剩余的一个bit X表示显示模式。

### 字符属性颜色位

X是1，前景色会比X是0时的前景色亮一些；背景色会闪烁。

![image-20210312111017073](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/note/image-20210312111017073.png)

### 疑问

设置显存的`nasm`代码

```assembly
mov		al,		'A'
mov		ah,		0Fh
mov		[gs:(80*3 + 20) * 2],		ax
```

`gs:(80*3 + 20) * 2`中的`3`是行坐标，最大值是24；`20`是列坐标。

上面的说法是正确的吗？

乘以2是什么意思？

## 又一个硬件---VGA模式的显示器

### 寄存器使用方法

读写硬件通过寄存器。使用VGA模式的显示器有6组显示器。设置光标位置、实现滚屏，只用到了一组寄存器：`CRT Controller Registers`。为了讲述方便，把它称作`R`。

R有两类寄存器，分别是`Adress Register`和`Data Registers`。

从名称就能看出，第一类寄存器只有一个，第二类寄存器有多个。

使用R的方法是，向`Data Registers`这组寄存器中的第`X`个寄存器写入数据。怎么设置`X`的值呢？通过向`Adress Register`写入数据设置`X`的值。

例如，要向`Data Registers`的第2个寄存器写入数据，需要先向`Adress Register`写入数据`2`。

### 寄存器详情

#### VGA寄存器

![image-20210312114435012](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/note/image-20210312114435012.png)



#### CRT Controller Registers

![image-20210312114513415](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/note/image-20210312114513415.png)



### 光标

通过设置`Cursor Location High Register`和`Cursor Location Low Register`的值控制光标的位置。

没弄清楚这两个值和光标位置的具体关系。

理解不了书上的代码（避免在小细节上浪费过多时间，不要和之前理解调度算法那样）。

### 滚屏

通过设置`Start Adress High Register`和`Start Adress Low Register`的值控制从显存的哪个位置开始显示。

这两个值和控制显存的显示位置的关系，没有弄明白。