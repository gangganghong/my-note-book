# 输入输出系统----键盘

## 代码位置

/Users/cg/data/code/os/pegasus-os/v25

## 回忆

### v1

这是一块难啃的硬骨头。一个多月前，我理解了、还写了文章。可现在几乎啥也不记得了。

这再次证明：理解了会记得更牢固，是不符合事实的。

即使是自己写出的东西，都会忘记。更不要提看的别人的东西了，例如，某某算法、某某java框架等。

依靠背诵知识点去面试，是非常不靠谱的。任你工作时间再长，若每次面试都需要依靠背诵知识点，那么，你每次遇到面试，都会像个新手一般去突击背诵。这很悲惨和心酸。

不说闲话。

尽快结束回忆，重新学习。

只记得几个关键词。最难的是解析算法。

#### 扫描码

键盘上的每个按键，对应一个或几个代号。例如，按下A键，计算机（分不清是CPU还是操作系统还是其他硬件）接收到的是123；松开A键，计算机接收到的是456。代号是我瞎写的。这样理解，同一个键，按下，计算机会接收到一个代号；松开，计算机又会接收到另一个代号。同一个按键，不同的动作，计算机接收到的代号，一般不相同。这个代号，叫“扫描码”。

不同年代，出现过几套不同的扫描码。我们正在使用的计算机键盘使用的是第二套扫描码。

一个按键，两种不同的动作会产生两类不同的扫描码，分别是：断码和通码。英语名称是什么？

断码和通码之间存在运算关系，这是人为设置的。

#### 两个硬件

忘记了这两个硬件的名字，暂且称呼它们是A、B吧。

> 13分钟。

1. 敲击或松开键盘按键时，A解析出对应的扫描码，发送给B。
2. B把解析码转换成第二套扫描码，通知8259A产生了键盘中断，并把新扫描码发送给8259A。

#### 和8259A的关系

8259A能设置放行或屏蔽键盘中断，能接收键盘数据。

键盘的硬件挂载到8259A的一个引脚上。

#### 解析算法

这应该是最大的难点，我完全不记得。

努力一下，让我解析，我会怎么处理。

两个硬件把按键解析成扫描码，键盘驱动（能这么叫吗）把扫描码解析成ASCII码。

建立一张ASCII码和扫描码的映射表，根据扫描码查询ASCII码。

映射表，例如，hashmap，键和值之间是映射关系。

用C如何建立haspmap？先创建一个结构体。它有两个成员，一个是key，一个value。再创建一个元素是结构体的数值。这就是一个时间复杂度为O(N)的hashmap。

上面写的是通用做法。针对扫描码和ASCII码这个具体场景，有其他方法吗？于上神用的方法就不是我上面想到的通用方法。

这个映射表是一维数组，元素是ASCII码。

key是数组索引。

key既是数组索引，又是扫描码。也就是说，要想一种方法，让扫描码成为数组索引。

扫描码是不是连贯的？如果不连贯，怎么能当数组的索引？

扫描码的初始值是不是0？如果不是，怎么当数组的索引？

> 我想不出怎么建立这个映射表，依靠残存的记忆也不行。耗费24分。

扫描码从某个值开始，在它前面的数组元素，全部用0填充。

##### Print Screen

组合键。

##### Pause

组合键

##### Enter

组合键。

##### 其他按键

只有一个扫描码，直接从映射表中找到值。

##### 组合键

不知道具体情况，不好处理。

回忆结束。

## 学习

### v1

#### 映射数组



![img](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/2e2eb9389b504fc28d62a7daeedde71190ef6d2b.jpg)



![查看源图像](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/d1160924ab18972b8d16b3c1e7cd7b899e510a88.jpg)



Make Code的顺序不是和按键顺序完全一致。对这一点，我确认了很久。

我知道了怎么建立映射数组。

1. 用键值除以3，商是行号，行号是Make Code。
2. 一个Make Code，在映射数组中找到键值是Make Code的元素，那个元素就对应按键上的基础值。
3. 一个Make Code + 加一个shift键的Make Code，在映射数组中找到键是（Make Code + 1)的元素，那个元素就是对应按键上的另一个值。
4. 不明白，于上神的映射数组中，一行有3个元素，第3个是什么？



我不打算直接复制于上神的映射数组，也不打算一个元素一个元素地敲击我的代码，那么，我打算怎么获得一份映射数组？

1. 映射数组的元素，每三个放到一行，行号是Make Code。
2. 有的按键会产生一个Make Code，有的按键会产生组合Make Code。
3. 在组合Make Code中，第一个字节是前缀。例如，Delete的按键产生的Make Code是`E0,53`。
4. 去掉前缀，留下剩余的字节。这个剩余的字节当作新Make Code。
5. Make Code是行号。再强调一次。
6. 接收到Make Code，可能是单一Make Code，也可能是组合Make Code。
7. 对于组合Make Code，去掉前缀，根据剩余的值确定该按键对应的值的行号。行号就是Make Code去掉前缀后的剩余部分。
8. 先定位行号，因为有前缀`E0`，因此，这个Make Code对应的是这一行的第三个元素。
9. 对于shift + A等组合键，也按相同的逻辑处理。差异是，对应的是这一行的第二个元素。

> 理解第三列、0x53--Delete（0x53行对应Delete)费了很多时间。受两份不同资料的误导。

http://www.vetra.com/scancodes.html

![image-20210418110445509](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210418110445509.png)



《一个操作系统的实现》第248页

![image-20210418110540685](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210418110540685.png)



```shell
/* 0x50 - Down		*/	PAD_DOWN,	'2',		DOWN,
/* 0x51 - PgDown	*/	PAD_PAGEDOWN,	'3',		PAGEDOWN,
/* 0x52 - Insert	*/	PAD_INS,	'0',		INSERT,
/* 0x53 - Delete	*/	PAD_DOT,	'.',		DELETE,
```

还剩下一个问题，怎么获取一份这个映射数组？逐个字母敲进去吗？

我决定，直接照抄于上神的映射表，但是，要完全理解他的映射表。

如果不照抄，我打算怎么做？找到一份比较原始资料，然后，解析网页，再建立我自己的映射表。

估计需要耗费两个多小时。我的主要目标是写操作系统，而不是解析HTML网页。解析HTML网页，没有价值。

等我实在有很多时间，想花大量时间看无用资讯的时候，可以解析一下。

找到了一份完整的 scan code set1表。地址是：http://users.utcluj.ro/~baruch/sie/labor/PS2/Scan_Codes_Set_1.htm。

> 浪费了太多时间。却并没有在操作系统知识上浪费时间。

### v2

#### 映射数组

1. 建立scan code set1的映射数组。
2. 这套编码的Make Code是连贯的，只是缺乏0。补齐0，就能建立映射数组，用Make Code当数组索引。
3. 在C语言中，数组天生就有索引，只不过，这个映射数组的索引，和字符有对应关系。
4. 一步步扩展这个数组。
5. Make Code 分为单一编码和复合编码。单一编码只有1个字节，复合编码有多个字节。
6. 先分析单一编码。
7. 假设，单一编码Make Code有N个。那么，从第0个元素到第N个元素，能建立一个数组。
8. 给出一个Make Code是Index，在数组中找到索引是Index的元素，这个元素就是Index对应的按键上的值。
9. 再分析复合编码。
10. 单一编码，是指按下一个键，产生的Make Code只有一个字节。复合编码，是指按下一个按键，产生的Make Code有多个字节。
11. 目前，分析的都是一个按键的情况。
12. 遇到复合编码，去掉前缀，剩余的字节，与单一编码的Make Code相同。
13. 这意味着，在目前的数组中，给出一个Make Code，有方法能查询到两个对应的值。
14. 目前，只能查询一个值。在当前元素的右边再增加一个元素，后面的元素依次往后移动。
15. 举个例子，假如，Make Code是5，对应的值是A，A的右边紧挨着小写字母a。
16. 假设，A的扫描码是5，a的扫描码是`e,5`。
17. 拿到扫描码N后，只有一个字节，直接找到数组中索引是N的元素。
18. 拿到的扫描码M，有前缀e，去掉前缀后是N。那么，去掉e，再找到数组中索引是N+1的元素。它就是M对应的值。
19. N对应的元素是arr[N]，M对应的元素是arr[N+1]。
20. 上面的查找方法是错误的。正确的应该是下面这样的。
21. 拿到Make Code后，Make Code乘以3才是数组的索引C。
22. 单一Make Code，索引就是C。
23. 复合编码，如果前缀是E，索引是C+2。
24. 复合编码，如果前缀是左右shift中的一个，索引是C+1。
25. 键盘上的按键，有的有两个值，一个是Base Value，另一个是Upper Value。
26. 键盘上的按键，并非每个按键的值都是可打印字符。不是可打印字符，无法在屏幕上输出。
27. 对这样的字符，在映射数组中，用自定义的代号代替。操作系统接收到这样的字符后如何处理是操作系统的事情。

#### 解析

##### 特殊键

###### Pause

###### Print Screen

0x1E -> 0001 1110

0X9E -> 1001 1110

0x80 -> 1000 0000

0x1E & 0x80 = 0x9E



0x4B = 0100 1011

接收到一个扫描码，怎么知道它是Make Code还是Break Code？

有的按键会产生多个扫描码，怎么接收完所有扫描码再开始判断？

检查第一个扫描码是不是E1？是，判断剩余的扫描码是不是1D、45、E1、9D、C5。不满足这个条件，这个扫描码是不可识别字符。

检查第一个扫描码是不是E0？是，判断剩余的扫描码是不是2A、E0、37。不满足这个条件，剥离E0，只检查剩余的扫描码，这就是这个按键在映射数组中的行号。

> 我只能想到这里了。

### v3

#### 缓冲区

敲击一个键后，一般会马上松开。这种场景，产生两个扫描码，一个是Make Code，另一个是Break Code。

同时按下shift键和A键，会产生四个扫描码，两个是Make Code，另外两个是Break Code。

有的键盘操作会产生更多扫描码。

我想说，只凭一个扫描码，不能识别出是那个键。需要缓冲区。*引出缓冲区的理由比较牵强，也许是因为我处理的场景太有限。*

缓冲区的结构：

```c
// 缓冲区的大小怎么确定？
#define KEYBOARD_BUFF_SIZE 128
typedef struct{
  		// 下一个空闲位置。存放数据到缓存区时，存储到这个成员。
			char *head;
  		// 下一个存储了数据的位置。从缓冲区获取数据时，从这个成员获取。
  		char *tail;
  		// 缓冲区存放的数据的数量。初始值是0。存储了一个数据后，这个值是1。
  		int counter;
  		// 存储到缓冲区的数据实际存储在这个数组。
  		// 初始状态，head、tail都指向这个数组的开始位置。
  		char buf[KEYBOARD_BUFF_SIZE];
}KeyboradBuffer;
```

#### 键盘中断例程

8042接收到8048送来的扫描码后会放进自己的缓冲区，等待中断例程来清空缓冲区。

中断例程从8042的缓冲区获取数据，放进自己的缓冲区。

```c

KeyboradBuffer keyboard_buffer;
// 初始化中断例程缓冲区
keyboard_buffer.head = keyboard_buffer.tail = keyboard_buffer.buf;
keyboard_buffer.counter = 0;

void keyboard_handler()
{
  		scan_code = read_from_8042_buff;
  		// while(keyboard_buffer.counter < KEYBOARD_BUFF_SIZE){
      if(keyboard_buffer.counter < KEYBOARD_BUFF_SIZE){
        	*(keyborad_buffer.head) = scan_code;
        	keyborad_buffer.head++;
        	keyboard_buffer.counter++;
        	if(keyboard_buffer.counter == KEYBOARD_BUFF_SIZE){
            	keyborad_buffer.head = keyborad_buffer.buf;
          }
      }
}
```

#### 读键盘中断例程的缓冲区

```c
void read_keyboard_buffer()
{
  		while(keyborad_buffer.counter > 0){
      		char scan_code = *(keyboard_buffer.tail);
        	// 解析扫描码的代码就在这里，不解析之前，只打印出扫描码
        	keyborad_buffer.counter--;
        	keyboard_buffer.tail++;
        	if(counter == 0){
          		keyboard_buffer.tail = keyboard_buffer.buf;
          }
      }
}
```

> 心算，不能透彻地想明白。自己写，不需要费力想就知道下一行应该写什么。

#### 开启键盘中断

> 上午的四个小时基本被浪费了，收益很小。原因是，花在回忆的时间太长了。

处理中断的流程是：

1. 建立中断例程
2. 设置8259A的ICW0，放行键盘中断。
3. 增加IDT项：
   1. 向量号是多少？
   2. 偏移量是键盘中断例程。
   3. 特权级是什么级别？和时钟中断的级别一致。1特权级，应理解为非0特权级。

理解对keyborad_buffer的操作应该具有原子性，耗费了许多时间。我仍没觉得这样做有必要。心算模拟不出不这样做的危害。

#### 从8402的缓冲区读数据

怎么读？

![image-20210418192113862](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210418192113862.png)

用汇编代码`in al, 60h`。

### v4

#### 解析扫描码

##### 回忆

这里的缓冲区是指中断例程的缓冲区。不说明，都是这个意思。

每次从缓冲区读一个字符，重点是，从缓冲区读。

根据第一个字符，

1. E0，先不处理。
2. 非E0，
   1. 是Make Code，在映射数组中，查找元素，arr[Make Code * 3]
   2. 是Break Code，不处理。
   3. 遇到shift和其他键怎么处理？
      1. 二者的扫描码出现的顺序是咋样的？

### v5

#### 解析shift+A组合键

##### 回忆

1. 在普通字符解析模块，检查当前扫描码是不是shift。设置识别扫描码是否为shift的变量is_shift。is_shift初始值是0。
2. 是，is_shift = 1。
3. 检查 当前扫描码 & is_shift == 1
   1. 满足这个条件，读取数组元素arr[MakeCode * MAP_CLOS + 1]，并设置is_shift = 0。
   2. 不满足这个条件，读取数组元素arr[MakeCode * MAP_CLOS]

##### 学习

于上神的代码太曲折，难理解。

我自己实现吧。

1. 检查扫描码是不是E0开头。若是，is_code_withE0 = 1。
2. 检查扫描码是不是shift。若是，is_shift = 1。
3. 检查扫描码是不是MakeCode。
   1. 是 & is_shift，元素是arr[MakeCode * MAP_CLOS + 1]，把is_shift设置为0。
   2. 是 & is_code_withE0，元素是arr[MakeCode * MAP_CLOS + 2]，把is_code_withE0设置为0。
4. is_code_with_e0、is_shift都是全局变量，在函数init_keyboard_handler中初始化。

> 45分。没理解于上神的代码，挺绕的。我自己想办法实现。

### v6

#### 解析HOME键等

> 看资料，耗费20分钟

把扫描码分为三类：

1. 以0xE1开头。
2. 以0xE0开头。
3. 除上面两种外的其他扫描码。

##### 0xE1

只有一个键，PAUSE。这个键只有Make Code。它的Make Code是：E1、1D、45、E1、9D、C5。

如果当前扫描码是0xE1，再从缓冲区中读取5个扫描码。每读取一次，就与上面的Make Code比较。一旦有一个不相等，就认为找个键不是PAUSE。如果全部相等，key设置为keymap.h中定义的常量PAUSE。

##### 0xE0

0xE0开头的键分为两类：

1. Print Screen
2. 其他0xE0开头的键。

Print Screen的扫描码：

1. Make Code：E0、2A、E0、37。
2. Break Code：E0、B7、E0、AA。

解析shift + A时，只要发现第一个扫描码是0xE0，就把is_e0设置成1。

现在，会分两种情况。

1. 解析当前的键是不是Print Screen。
   1. 检查是不是这个键的Make Code。
   2. 检查是不是整个键的Break Code。
   3. 如果是，怎么处理？
2. 如果不是Print Screen，把is_e0设置成1。

##### 其他

在这个分支打印字符。

不是左右shift && 是Make Code，就打印。这是我解析shift + A时打印字符的条件。

排除了Pause和Print Screen，可没有排除Home这些键。怎么排除Home这类键？

所谓排除Home这类键，只是不打印这类键。方法是：在这类键的原始常量值加上一个数，让它们的第3位是1。所有按键的值一定是一个字节，把那些不可打印字符设置成12个bit。只需检查一个字符的第3位是不是1就能识别出它是不是不可打印字符。

把打印字符的功能放到函数in_process中完成。这个函数除了打印字符，可能还会实现Enter键换行功能等。

要设置哪些字符不可打印，只需在它们的常量值上加上0x0100即可。

识别出扫描码是shift、ctrl时，key是arr[MAP_COLS * Make Code]。

> 22分

## 写代码

### 缓冲区

从8402中读取数据到中断例程的缓冲区。

1. ~~新增文件keyboard.c、tty.c。尽量用更少的文件。~~
2. 新增一个进程，
   1. 在proc_table中新增，进程体在kernel_main.c中，名称是task_tty。task_tty只执行keyborad_read()。
   2. 在task_table中新增一个元素。
3. ~~在keyborad.c中放置一切和键盘有关的代码。~~
   1. ~~Keyboard_read，从中断例程的缓冲区中读取数据。~~
   2. ~~先不管代码安排。~~
4. 算了，尽量全部放在kernel_main.c中吧。
5. 新建KeyboardBuffer结构，作为缓冲区。
6. 修改keyboard_handler函数，把数据读取中断例程的缓冲区。
7. 在kernel_main中初始化keyboard_buffer。

### 解析普通按键

0x1E---> 0001 1110

0x9E---> 1001 1110

0x7F---> 0111 1111

最高位是0，就是Make Code。

怎么检查数字的最高位？

1000 0000--->80

> 又纠结了很久。看书吧。

判断方法是：检查 Make Code & 0x80 的结果。结果是0，是Make Code；结果是1，是Break Code。

我想到的方法是正确的，只是觉得这和Make Code经过运算获得Break Code相同而觉得是错误的。其实，计算一下就知道了。

1. 新建keymap.h，把于上神的映射数组复制进去，并自己建立相关常量。
   1. 怎么放到Makefile中？
2. 剥离读取缓冲区的代码，封装成函数read_from_keyboard_buf。
3. 修改keyboard_read

> 比较顺利。

又出现8402的缓冲区容易满大的问题。之前修改调度代码、让TTY总是执行后，这个问题消失了。现在又出现了，可能是“剥离读取缓冲区的代码，封装成函数read_from_keyboard_buf”造成的。

### shift + A

1. 建立全局变量is_code_with_e0，扫描码是不是0xe0。
2. 建立全局变量is_shift，扫描码是不是shift。
3. 建立变量is_disp，是否打印，1打印，0不打印；初始值1。
4. 建立shift_l、shift_r等常量，常量值是它们的Make Code。
5. 建立函数init_keyboard_handler，初始化is_shift、is_code_with_e0等全局变量。
6. 在kernel_main函数中调用init_keyboard_handler。
7. 在keyboard_read中，
   1. 非0xe1、非0xe0模块，
      1. 如果扫描码是shift，is_shift = 1，is_disp = 0。
      2. 如果扫描码是Make Code
         1. 若is_shift = 1，元素是 arr[MakeCode + 1]，is_shift = 0。
         2. 若is_code_with_e0 = 1，元素是arr[MakeCode + 2]，is_code_with_e0 = 0。
         3. 其他，元素是 arr[MakeCode ]
      3. 若is_disp是1，打印字符。若is_disp是0，不打印。
      4. 在最后，把is_disp设置为1。

> 想思路，16分钟。

> 调试21分，毫无头绪。

### 解析HOME键等

实际是解析所有键。

1. 在keymap.h中新建常量 FLAG_EXT，值是0x0100。把keymap.h中的不可打印字符的常量值加上FLAG_EXT。
2. 在main.c中
   1. 新建函数，in_process，把之前打印字符的操作放到这个函数中。这个函数以后可能会完成其他功能，例如，实现换行键。
   2. 补充函数keyboard_handler
      1. 0xE1分支，识别Pause
      2. 0xE0分支，
         1. 识别Print Screen，按下
         2. 识别Print Screen，松开
         3. 其他复合类型的扫描码，设置is_e0 = 1
      3. 不是Pause，也不是Print Screen
         1. 在原来的计算上做小修改。
         2. 最大变动，是用in_process替换原来的打印字符串代码。
         3. `in_process`的参数是key。
         4. key是按键在映射数组中的值。

> 10分钟。

> 写代码花了43分钟。需要思考的地方不多，慢悠悠写，就花这么多时间。

## 调试

### 缓冲区

```shell
overflow in conversion from 'long int' to 'char' changes value from '68719476638' to '-98'
```

![image-20210418225444753](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/lab/image-20210418225444753.png)

按下A键，读取的Break Code 是`0xFFFFFF9E`，和正确的Break Code比，多了前面的6个F。

在in_byte函数中，从20h读取到的数据没有错误，进入keyboard_handler后，变成错误数据。

in_byte执行这句后造成的，`movsx eax, byte ptr ss:[ebp-13]`。

花了一个多小时，知道了错误是什么？char scan_code 赋值为0x9E，溢出了，变成负数。改成unsigned char scan_code，问题消失。

缓存区写满或读完的判断条件，不能是counter。

还有一个错误，几个键盘操作就会出现Invalid Code错误。<------ 跳转到 restore 使用了jnz，改为jmp后错误消失。

##### 长按A键

长按任何一个按键不松开，键盘中断程序就没有被运行。

```shell
01168784000i[KBD   ] internal keyboard buffer full, ignoring scancode.(25)
01170348000i[KBD   ] internal keyboard buffer full, ignoring scancode.(a5)
01170348000i[KBD   ] internal keyboard buffer full, ignoring scancode.(25)
01171824000i[KBD   ] internal keyboard buffer full, ignoring scancode.(a5)
01171824000i[KBD   ] internal keyboard buffer full, ignoring scancode.(25)
```

找到原因了，调度程序出了问题。

先猜想，再验证。这个调试速度，我基本满意。

#### movsx

汇编语言[数据传送](https://baike.baidu.com/item/数据传送/500685)指令MOV的变体。带符号扩展，并传送。

例如：

1.MOV BL,80H

MOVSX AX,BL

运行完以上汇编语句之后，AX的值为FF80H。由于BL为80H=1000 0000，最高位也即符号位为1，在进行带符号扩展时，其扩展的高8位均为1，故赋值AX为1111 1111 1000 0000，即AX=FF80H。

2.mov CL, 50H

MOVSX AX, CL

50H=0101 0000，最高位为0，则AX为0000 0000 0101 0000

结果AX = 50H

> 消耗时间最多的是：in_byte读取到的0x9E变成了0xFFFFFF9E。试了非常多次，又陷入低效调试，手指都按疼了。
>
> 问题出在，keyboard_handler嵌套调用in_byte。

#### gdb

删除断点

```shell
(gdb) info break
Num     Type           Disp Enb Address    What
1       breakpoint     keep y   0x00031245 in keyboard_handler at kernel_main.c:841
2       breakpoint     keep y   0x000312bd in keyboard_read at kernel_main.c:861
	breakpoint already hit 2 times
(gdb) clear keyboard_read
No breakpoint at keyboard_read.
(gdb) info break
Num     Type           Disp Enb Address    What
1       breakpoint     keep y   0x00031245 in keyboard_handler at kernel_main.c:841
2       breakpoint     keep y   0x000312bd in keyboard_read at kernel_main.c:861
	breakpoint already hit 2 times
(gdb) clear 2
No breakpoint at 2.
(gdb) info break
Num     Type           Disp Enb Address    What
1       breakpoint     keep y   0x00031245 in keyboard_handler at kernel_main.c:841
2       breakpoint     keep y   0x000312bd in keyboard_read at kernel_main.c:861
	breakpoint already hit 2 times
(gdb) info b
Num     Type           Disp Enb Address    What
1       breakpoint     keep y   0x00031245 in keyboard_handler at kernel_main.c:841
2       breakpoint     keep y   0x000312bd in keyboard_read at kernel_main.c:861
	breakpoint already hit 2 times
(gdb) delete 2
(gdb) info b
Num     Type           Disp Enb Address    What
1       breakpoint     keep y   0x00031245 in keyboard_handler at kernel_main.c:841
```

### Shift + A

解决了问题，从按下A键后cols = 2入手。原来，是在init_keyboard_handler函数中重新声明并赋值is_e0，导致is_e0的初始值不为0。

> 无头苍蝇、期望改一点然后运行看看行不行来调试，太浪费时间，太累，太低效。

## 总结

昨天消耗时间最多的是：回忆知识点。

今天消耗时间最多的是：

1. 8402的缓冲区很快就满了。
2. 输入几个字符就出现Invalid Code。
3. 解析shift + A。

解决前两个问题，我还比较满意。

从14点到18：47，一直在处理shift + A。消耗时间最多的是调试。低效、没有仔细思考。

1. 基本思路，没有问题。
2. 写的代码，有多处漏洞，不完全符合我的思路。
3. 在函数内又重新创建了同名变量，导致全局变量没有初始化。

