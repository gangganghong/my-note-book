# 键盘阅读笔记

## 开启键盘

伪代码

```c
// 处理键盘输入
void keyboard_handler()
{
  		// 有输入就打印星号
  		disp_str("*");
}

// 初始化键盘输入配置
void	init_keyboard()
{
  		// 配置键盘输入的中断向量和对应的中断例程
  		put_irq(KEY_BOARD_VECTOR,keyboard_handler);
  		// 开启键盘输入中断监听
  		enable_irq(KEY_BOARD_VECTOR);
}
```

## 键盘种类

1. AT。古老。
2. PS/2。
3. USB。

## 流程

1. 敲击键盘。
2. `8048`获取键盘操作对应的扫描码，发送给`8042`。
3. `8042`把从`8048`获取到的扫描码转成第一套扫描码，放入缓冲区。
4. `8042`通知`8259A`发生中断，中断处理程序会取走缓冲区的数据。
5. 缓冲区的数据没有被取走，`8042`不会再接收新数据。

书上写明了这个流程，我却看其他博客和外国网站，徒劳无功。

知道了这个流程又怎样？过年前我就知道了，现在看却像是新知识。

我写的博客，都是我的理解，不是纯粹的摘录，我不信任它们，准确地说，未来不能作为资料去查找。

其实，可以解决这个问题。对照资料，更正不准确的点。

## 零散笔记

有三套扫描码（scan code set 1、scan code set 2、scan code set 3）。

每次键盘操作会产生两个扫描码：

1. Make Code。按下键或按下键并且一直按住。
2. Break Code。松开键、键弹起的时候。

除`Pause`键外，敲击其他键都会产生`Make Code`和`Break Code`。

`Break Code = Make Code | 0x80`。

scan_code & 0x80 

1. ~~结果是true，那么，scan_code是Make Code。~~
2. ~~结果是false，那么，scan_code是Break Code。~~

只能举例子。

Scan Code 是 0x1E，0x1E & 0x80 的结果是，0001 1110 & 1000 0000 = 0，

Scan Code 是 0x9E，0x9E & 0x80 的结果是，1001 1110 & 1000 0000 = 1，

1. ~~结果是false，那么，scan_code是Make Code。~~
2. ~~结果是true，那么，scan_code是Break Code。~~

1. res = 0001 1110 & 1000 0000 = 0，make = res ? false:true。
2. 注意这个`?:`语句。res是true(Break Code)，make是false；res是false(Make Code)，make是true。

理解运算"|"和"&"。

`0x03 | 0x80`，结果是，`0x83`。

`0x13 | 0x80`，`0001 0011 | 1000 000`，结果是，`1001 0011 = 93`。

与`0x80`进行“或”运算后，再与`0x80`进行“与”运算，结果一定是非0。因为，第四位数是1。

实际运行代码，能解决许多问题。

## 调试

/home/cg/yuyuan-os/osfs06/q

/home/cg/yuyuan-os/osfs07/d

cp -rvf /home/cg/yuyuan-os/osfs06/q/bochsrc* ./

```shell
/home/cg/tools/bochs-2.6.11/bin/bochs -f bochsrc-gdb 
b keyboard.c:59
target remote:1234
```



97 & 0x0100

97 = 16*6 + 1

0x61 & 0x0100 = 0x00

## ASCII

![img](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/Center.gif)





![img](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/Center.png)



扩展ASCII码

![img](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/Center-20210311180136421.gif)

