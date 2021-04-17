# 实现进程---延迟函数和进程调度

## 代码位置

/home/cg/os/pegasus-os/v24

## 回忆

### v1

#### 延迟函数

通过对8253编程，能设置时钟中断频率，即让时钟中断每隔多少个时间单位发生一次。

 我已经写了一个延迟函数，循环 time * 10 *10000次。消耗了多少时间，不知道。

要写一个新延迟函数，能精准地控制延迟多少个时间单位，例如，延迟N微秒。

不是很费劲，我能很快想出怎么写这个函数。伪代码如下：

```python
def millisecond_delay(n)
		// 进入函数时，已经发生的时钟中断次数
		init_ticks = get_ticks()
    // 怎么确定这个计算公式？
  	need_tick_num = 使用n计算出需要多少个时钟中断
    // 当前时钟中断次数-进入函数时已经发生的时钟中断次数=本函数已经运行的时钟中断次数N
    // 当N <= need_tick_num 时，继续运行循环
  	while((get_ticks() - init_ticks) <= need_tick_num){
      	// do nothing
    }
```

唯一的难点，根据微秒数计算出时钟中断次数。

> 微秒，时间单位，符号μs（英语：microsecond ），1微秒等于百万分之一秒（10的负6次方秒），1毫秒等于千分之一秒（10的负3次方秒）。

如果能把时钟中断设置成每微秒发生一次，就好了。

能设置吗？不知道。

每微秒发生一次，1秒发生一百万次。8253无法设置1秒钟发生这么多次。

每100微秒发生一次，1秒发生1万次。

> 有点想不明白。不继续下去了。

#### 进程调度

1. 每个进程设置一个ticks、一个priority。
   1. 每次被调度到，这个进程的ticks减去1。
      1. ticks等于0的进程，会被调度到吗？
      2. 
   2. priority的值不变。

##### 调度算法

遍历存储进程表的数组，每个进程表的ticks都和某个值比较。

想不起来了。直接学习吧。

## 学习

### 延迟函数

#### v1

8253在N个时间单位内产生1193180次时钟中断，时间单位是秒。

实现1microsecond产生一次时钟中断，即1秒钟产生10的6次方时钟中断，应该满足`1193180/8253的计算器的值 = 1`。

> 费了点功夫才想明白。

8253的counter0计数器是16位的，最大值是65535，不能设置为1193180。所以，不能实现1微秒产生一次时钟中断。

counter0的时间单位是秒，

1. (counter0 * 10的6次方) /1193180 = 每次时钟中断消耗1微秒。~~做不到。~~ 能做到。counter0的值约等于？
2. (counter0 * 10的6次方) /1193180 = 每次时钟中断消耗10微秒。

> 想不明白。浪费时间。直接看书吧。

> 1秒 = 1000毫秒
> 1毫秒 = 1000微秒
> 1微秒 = 1000纳秒
> 1纳秒 = 1000皮秒
> 1s = 1000ms
> 1ms = 1000μs
> 1μs = 1000ns
> 1ns = 1000ps
> 1秒(s) =1000 毫秒(ms) = 1,000,000 微秒(μs) = 1,000,000,000 纳秒(ns) = 1,000,000,000,000 皮秒(ps)=1,000,000,000,000,000飞秒(fs)=1,000,000,000,000,000,000仄秒(zs) =1,000,000,000,000,,000,000,000幺秒(ys)1,000,000,000,000,000,000,000,000渺秒(as)

想让时钟中断每10ms发生一次，即，让时钟中断每1秒发生100次。也就是，让1193180次时钟中断在N秒完成。

N * 100 = 1193180。

counter0 = 1193180 / 100，约等于 11931。

如何设置counter0的值是11931？又是一个已经掌握过的难题，但我已经忘记了。详情请查看：`/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/lab-6.md`。

##### 设置8253的counter0

详情在《一个操作系统的实现》6.5.2--->6.5.2.1

![image-20210417160109621](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210417160109621.png)



![image-20210417160137511](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210417160137511.png)



![image-20210417160210103](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210417160210103.png)



![image-20210417160233547](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210417160233547.png)



![image-20210417160253382](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210417160253382.png)



![image-20210417160316238](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210417160316238.png)

#### v2

理解延迟函数费了很大劲。

```c
void milli_deay(int milli_sec)
{
			int t = get_ticks();
  		while((get_ticks() - t)/每秒时钟中断次数 * 1000) < milli_sec){}

}
```

1. `get_ticks() - t`，已经运行了的时钟中断次数N。
2. `N / 每秒时钟中断次数`，消耗了多少秒M。
3. `M * 1000`，把秒转换成为微秒。

## 写代码

### 延迟函数

1. 在kernel_main.c系统调用下方声明函数：`void milli_delay(int milli_sec)`。
2. 在kernel_main.c的最下面创建函数`void milli_delay(int milli_sec)`。
3. 分别在进程A、B、C中加入：milli_delay(10)、milli_delay(20)、milli_delay(30)。

> 不费吹灰之力，7分钟写完。

## 调试

### 延迟函数

```shell
00014033826i[BIOS  ] Booting from 0000:7c00
00020676364e[CPU0  ] write_virtual_checks(): write beyond limit, r/w
00020786442i[CPU0  ] WARNING: HLT instruction with IF=0!
```

为什么会出现这个问题？

先直接说答案：显存耗尽了。

在`get_ticks`函数中，我打印了两个字符。`milli_delay`每循环一次都会调用一次`get_ticks`，很快就耗尽显存。

在100次时钟中断中，get_ticks运行了不知道多少次。

怎么发现问题的？

1. 怀疑是milli_delay引起的。
2. 对照于上神的代码。我的代码和他的代码并无二致。
3. 断点调试，发现屏幕上极速出现的字符，我联想到”显存耗尽“。
4. 修改之后，果然如此。

解决了问题，运行效果如图：

![image-20210417165427521](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210417165427521.png)

我看不出milli_delay是否发挥了作用。

> 消耗时间30分钟，还算正常。

## 总结

### 延迟函数

#### 小结

1. 回忆计数器的计算方法、重看资料后理解计算方法，消耗了不少时间。不应该啊。
2. 理解延迟函数的比较方法
   1. 我的思路，把延迟的时间换算成时钟中断次数。
   2. 于上神的思路，把时钟中断次数换算成微秒数。
   3. 两种思路，都可以。我理解于上神的思路，又纠结自己的思路，思考过程不清晰。
   4. 于上神的思路：
      1. 轮询，获取当前ticks，减去进入延迟函数时的ticks，等于消耗的时钟中断次数。
      2. 时钟中断次数除以时钟中断频率（每秒发送的时钟中断次数），等于，这些时钟中断次数消耗的秒数。
      3. 秒数乘以1000，等于微秒。
      4. 判断条件，不应该包含等于。一旦等于，就已经运行了需要延迟的微秒数了，需立即结束延迟。
3. 显存耗尽。找到问题的时间，大概还正常。花了30分钟。太多了。反复运行，灵光乍现，靠直觉解决问题。

#### 遗留问题

我不知道怎么看是否的确延迟了1秒。