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

### v2

设计tty框架。

```c
typedef struct{
  	// 控制台的当前位置
  	unsigned int current_start_addr;
  	// 控制台在显存中的位置
  	unsigned int addr_in_vm;
  	// 控制台占用的显存的大小
  	unsigned int limit_in_vm;
  	// 鼠标的当前位置
  	unsigned int cursor;
}Console;
```

Console是什么？能看到吗？

在我的操作系统中，呈现在眼前的屏幕就是控制台，敲击键盘，会显示字符。这就是控制台的外在形式。

在80*25模式下，显存一共有32KB。每个控制台至少需要 `80*25*2 = 4000`个字节。32KB足够划分成8个左右控制台。

每个控制台占用一块显存。这块显存，有初始地址，有长度。初始地址就是addr_in_vm，长度是limit_in_vm。

current_start_addr是控制台的当前位置。这是什么？每个控制台至少需要`80*25*2=4000`个字节，但是，一个控制台可能有8000个字节、16000个字节。如果一个控制台占用的显存大小是16000个字节，并且，它的初始地址是0。那么，current_start_addr可能是0、4000、8000、12000。我认为，current_start_addr，是addr_in_vm + 4000*(N-1)，N是控制台的第几屏。

> 学习TTY这章，效率非常低。看不懂许多细节，越看不懂越容易分心。
>
> 我找到了方法。遇到这样的学习内容，看一点，就复述一点，而不是像以前那样看完所有内容再复述。
>
> 尽量弄清楚所有细节，然后记住。
>
> 实在弄不明白的，就搁置。

```c
#define TTY_BUFFER_SIZE 1280

typedef struct{
  	unsigned char *tail;
  	unsigned char *head;
  	unsigned int counter;
  	unsigned char buf[TTY_BUFFER_SIZE]
    // 控制台  
   	Console *console;
}TTY;
```

在讲解Console时，文字部分提到的终端，不是Console，应该是TTY。

TTY，由一个缓冲区和一个CONSOLE结构组成。缓冲区和键盘缓冲区结构一致。TTY就是在操作系统中看到的那个屏幕，通过ALT+FN切换。从键盘缓冲区中读取数据、放入TTY的缓冲区，再从TTY的缓冲区中读取数据写入显存。

```c
#include <stdio.h>

int main(int argc, char **argv)
{
        int arr[3] = {0,1,2};

        printf("arr[0] = %d\n", arr[0]);

        int *ptr = arr;
        printf("arr[0] = %d\n", *ptr);
        ptr++;
        printf("arr[0] = %d\n", *ptr);
        printf("arr[2] - arr = %d\n", &arr[2] - arr);


        typedef struct{
                int age;
                int height;
        }Person;

        Person arr2[3];

        Person p1 = {23, 24};
        Person p2 = {25, 26};
        Person p3 = {27, 28};

        arr2[0] = p1;
        arr2[1] = p2;
        arr2[2] = p3;

        printf("arr2[2] - arr2 = %d\n", &arr2[2] - arr2);

        return 0;
}
```

```shell
[root@localhost c]# ./array
arr[0] = 0
arr[0] = 0
arr[0] = 1
arr[2] - arr = 2
arr2[2] - arr2 = 2
```

用GCC把C代码编译成汇编代码，命令是：`gcc -S arr.c`。编译结果是as汇编。

arr是一个数组，&arr[2] - arr = 2。理解不了。为啥不是32？

> 耗费时间太多了。时间跨度大概3天，实际用时10多个小时。
>
> 分心，不想看，理解不一些细节。

### v3

> 一举终结TTY的理论知识，已经耗费了太多时间，不能再拖了。

#### 数据结构

先从两个结构体说起。

TTY

```c
typedef struct{
  	// 当前光标位置
  	unsigned int cursor;
  	// 当前终端的在显存中的初始地址
  	unsigned int vm_original_addr;
  	// 当前终端的大小
  	unsigned int vm_limit;
  	// 当前终端的屏幕的初始地址，第几屏。用文字不好描述，用图却很容易说明
  	unsigned int vm_start_video_addr;
}CONSOLE;
```

TTY的console

```c
#define TTY_BUFFER_SIZE 128

typedef struct{
  	unsigned char *head;
  	unsigned char *tail;
  	unsigned int counter;
  	unsigned char tty_buffer[TTY_BUFFER_SIZE];
  
  	CONSOLE *s_console;
}TTY;
```



> 不知道怎么用连接词。那就不用了。

#### 进程体

```c
void TTY()
{
  	// 初始化所有TTY
  	init_tty();
  
  	while(1){
      	// 遍历所有tty start
      	// 从键盘缓冲区把数据读取到TTY的缓冲区
      	tty_do_read();
      	// 从TTY的缓冲区把数据写入到显存
      	tty_do_write();
      
      	// 遍历所有tty end
    }
}
```

#### 初始化一个tty

```c
TTY tty;
tty.head = tty.tail = tty.tty_buffer;
tty.counter = 0;

unsigned int index = tty_cur - tty_table;
tty.console = *console_table[index];
```

#### tty_do_read

遍历tty数组时，如果tty是当前tty，从键盘缓冲区读取数据到tty缓冲区。

当然，可以按上面的思路去实现。但是，于上神不是这样设计的。他的思路是这样的。

```c
if(如果tty是当前tty){
		keyboard_read();
}
```

#### tty_do_write

把tty缓冲区的数据写入显存。

这不是一个复杂的问题。只需看一个字母，我就能想起全部思路。这是在背诵代码吗？

从tty缓冲区读取字符，然后写入显存。tty_do_write本身不包含循环，它本身在一个循环体中。tty_do_write本身只做两件事：

1. 从tty缓冲区读取一个字符。
2. 把这个字符写入到显存。

怎么从缓冲区读取字符？

1. 前提条件是，缓冲区还有数据，即，缓冲区的counter大于0。
2. 从tail读取数据。
3. 把tail往前移动一位。
4. counter++。
5. 如果tail已经在缓冲区的最后一个位置后面，重置tail为缓冲区的开头位置。

> 写出了这些，就已经算掌握了tty_do_write吧。
>
> 不写伪代码了。

> 为什么我觉得tty这块内容比较烦？
>
> 其他的章节，看一次，我能记住大部分内容，还能把这些内容条理化。
>
> tty这块，比较琐碎，细节比较多。我没有能力把它们全部记住，逐个记忆，又觉得没有真正掌握。

#### init_screen

> 好烦啊。不想看了。躺在沙发上写这些东西，尽量让自己的“烦”少一点。
>
> 其实，我很想让自己正襟危坐写东西。

这个函数，就是给console结构体的每个成员赋值而已。

##### cursor

光标位置。

如果console是第0个，那么，光标的位置沿用之前的dis_pos。dis_pos和光标有什么关系？cursor = dis_pos / 2。

光标的位置必须沿用之前的dis_pos吗？不是的。完全能够把光标的位置和dis_pos的位置都设置成0。

如果console不是第0个，那么，光标的位置是多少？不需要直接设置，而是用out_char打印两个字符`#`、`序号`。out_char会自动移动cursor的位置。

##### vm_limit

每个终端都占用一定长度的显存。怎么确定一个终端占用显存的大小？我的方法是，把显存等分为N份。N是终端的数量。

##### original_addr

每个终端都从显存的某个位置开始。

上面那句，还没有解释完。

第0个console的original_addr：0。

第1个console的original_addr：vm_limit。

第2个console的original_addr：vm_limit * 2。

第N个console的original_addr：vm_limit * N。

##### start_video_addr

在我使用的bochs的屏幕中，假设在整个屏幕都写满字符，占用A字节字符。一个终端的vm_limit足够显示几个屏幕的字符。每个屏幕开始位置对应的显存地址，就是start_video_addr。start_video_addr是A的倍数。

所有console的初始start_video_addr的值都是original_addr。

##### flush

把上面的数值写入到两组寄存器：

1. 设置光标的寄存器。
2. 设置start_video_addr的寄存器。

寄存器的使用方法：

1. 向地址寄存器写入索引，选择具体要使用的那个寄存器。
2. 向数据寄存器写入数据，设置start_video_addr和光标的位置。

#### out_char

`out_char(CONSOLE *console, unsigned char ch)`。

打印一个字符，用来代替已有的`disp_str`。为啥要新建一个功能相同的函数？我不知道。

向显存写入新字符，前提是，还有剩余的显存。

所有操作的内存地址的单位都是字，光标的移动单位也是字。

公共条件：

1. #define VM_BASE 0xB8000
2. 写入下一个字符的显存的位置：next_char_vm_addr。

##### 普通字符

最后一个字符的显存地址，小于当前终端的最大地址。

1. *next_char_vm_addr++ = ch
2. *next_char_vm_addr++= colour_value

##### \b

退格键。

最后一个字符的显存地址，大于当前终端的original_addr。

退格，就是把上一个字符改成空格。

1. *(next_char_vm_addr-2) = ch;
2. *(next_char_vm_adr-1) = colour_value;

##### \n

换行键。

最后一个字符的显存地址，小于倒数第二行的末尾地址。

理解换行的计算方法，耗费了海量时间。不是说理解换行算法本身难，而是解释如何提出这个算法困难。

我懒得再耗费海量时间了。就是这么回事：

1. 统计行数的方法，基数是1，而不是0。
2. 从终端在显存的初始位置到当前字符，一共使用了多少显存？
3. 用使用的显存数量除以每行占用的显存长度。结果是，当前字符所在的行的序号。例如：
   1. 在第1行的字符，商是0，当前行的行号是1。
   2. 在第1行的字符，这个字符是这一行的最后一个字符，商仍然是0.
      1. 为什么？
      2. 假如，每行的长度是WIDTH个字，那么，最后一个字符的位置是WIDTH-1。
      3. 因为，每行的长度的基址是0。
   3. 终于要说到换行了。
   4. 当前字符所在的行号等于 = 当前字符在显存的位置（用字作为单位）/ WIDTH + 1。
   5. 换行，就是移动到当前字符所在的行的下一行。
   6. 也可以这么理解，目标行相对于显存的初始地址的偏移量 = 当前字符所在的行的行号 *  WIDTH。
   7. 最终结果是：目标行的偏移量 = ((当前字符在显存的位置 / WIDHT) + 1)*WIDTH.
   8. 如果光标的位置大于一屏，向下滚屏。
      1. cursor >= current_start_addr + SCREEN_WIDTH
      2. current_start_adr是每凭的初始地址。

> 果然，在天才看来，怎么换行？他能直接想到最终结果。而我，看到了最终结果，还需要费劲地推想用什么方法能得到最终结果。
>
> 我果然只是一个平凡的人。

#### scroll_screen

滚屏。

显示器的屏幕有高度，填满这个高度的屏幕之后，终端的显存仍没有被使用完。如果不改变屏幕的初始显存地址，即使继续向终端的显存写数据，也不会在显示器上看到字符。

怎么办？只能滚屏。

滚屏，需要有空间可以滚。向上滚，前提是，当前光标在大于第一屏的那一屏；向下滚，前提是，当前光标所在的那一屏不是最后一屏。

怎么判断当前光标所在的那一屏大于第一屏？

每一屏的初始地址是current_start_video，第一屏的current_start_video = original_addr，第二屏的current_start_video = original_addr + SCREEN_VM。

如果current_start_video - original_addr > 0，那么，可以往上滚屏。

怎么判断能不能往下滚屏？

上面的方法是错误的。当终端的剩余显存大于一屏所需要的显存，才能往下滚屏。

如果  cursor + SCREEN_VM  < original_addr + limit，能往下滚屏。

#### in_process

完成的功能：

1. 把可见字符放入tty的缓冲区。
2. 把换行符、退格符转成\n、\b放入tty的缓冲区。
3. 处理上下滚屏、终端切换键。

#### put_key

`put(TTY *tty, unsigned int key)`，把字符放入tty的缓冲区。流程如下：

1. 缓冲区没有满，才放入新字符。
2. 把新字符放到head，同时head往前移动一位。
3. couter++。
4. 如果head == buf + 缓冲区的长度，初始化 head = buf。

#### tty_do_read

如果是当前终端，执行 keyboard_read。

#### tty_do_write

从tty的缓冲区取出数据，用out_char写入缓冲区。流程如下：

1. 缓冲区还有数据，才取出数据。counter > 0。
2. 从tail取出数据，然后tail往后加1。
3. counter--。
4. 如果tail == buf + 缓冲区的长度，初始化tail = buf。

> 对照代码，为代码的逐个细节写了我自己的理解的注释。
>
> 我认为，对tty的学习，就应该结束了。
>
> 我非常不乐意在写TTY代码前再度先写出它的代码细节再写代码。

## 写代码

在main.c中新增：

1. TTY结构。
2. CONSOLE结构。在TTY结构前面。
3. #define TTY_NUM 3
4. TTY tty_table[TTY_NUM]。
5. CONSOLE console_table[TTY_NUM]。
6. TTY的进程体TastTTY
   1. Init_tty
      1. 一个循环
         1. 初始化缓冲区。
         2. init_screen
            1. 初始化console的每个成员。
   2. 设置当前tty，select_console(0)。
   3. 一个循环
      1. tty_do_read
         1. 从键盘缓冲区把数据读到tty缓冲区，使用put_key
            1. put_key(unsigned int key)，把数据放入tty的缓冲区。
      2. tty_do_write
         1. 从tty缓冲区读取数据，交给in_process处理
         2. in_process：
            1. 可见字符，打印,out_char(key)
            2. 不可见字符
               1. ENTER，转成\n,out_char("\n");
               2. BS，转成\b,out_char("\b");
               3. ALT + FN，切换控制台
               4. SHIFT  + UP，向上滚屏，scroll_up
               5. SHIFT + DOWN，向下滚屏，scroll_down
            3. 怎么设计，是我的自由，能实现功能，又能兼容已有代码即可，无需束手束脚。
7. 上面都是属于tty的代码，下面是console的代码。
8. out_char(TTY *tty, unsigned int ch);
   1. ch，我觉得应该是unsigned char类型。
9. select_console(unsigned char console_index)
10. scroll_up
11. scroll_down

> 46分钟。

> 花了3个小时，写完了上面文字描述的代码。out_char、换行是难点。
>
> 不，还没有写完。in_process需要改造。
>
> 改造了in_process，不知道还有没有其他错误。耗时28分钟。
>
> 修正语法错误，耗费1个小时。编译，然后根据报错信息逐条修改：
>
> 1. 忘记加分号。
> 2. out_char中的当前位置没有写成指针类型。
> 3. 弄错结构体的成员变量。
> 4. int 和 char 类型不一致。
>
> 断点调试28分钟，毫无线索。
>
> 总计耗费时间：6小时50分。
>
> 错误原因是：根据console的光标地址计算绝对地址时重复加了显存基地址。
>
> 怎么解决问题的？断点调试，定位到输出有问题，进一步定位到计算显存绝对地址有问题。
>
> 为啥会耗费这么多时间？TTY进程和键盘中断、时钟中断混合在一起，比较难理清执行流程，比较难创造测试条件。
>
> 后来怎么创造条件的？只运行TTY进程。
>
> 还有什么问题？
>
> 退格时，光标消失。小小修改后，测试了很多次，没有任何收获。

设置光标时，值不是绝对显存地址，即，不需要也不能加上0xB8000这个基址。



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

## 调试

```shell
main.c: In function 'put_key':
main.c:1214:16: warning: comparison of distinct pointer types lacks a cast
   if(tty->head == tty->buf + KEYBOARD_BUF_SIZE){
                ^~
main.c:1215:14: warning: assignment to 'unsigned int *' from incompatible pointer type 'unsigned char *' [-Wincompatible-pointer-types]
    tty->head = tty->buf;
              ^
main.c: In function 'out_char':
main.c:1270:5: error: invalid type argument of unary '*' (have 'unsigned int')
     *(addr_in_vm - 2) = ' ';
     ^~~~~~~~~~~~~~~~~
main.c:1271:5: error: invalid type argument of unary '*' (have 'unsigned int')
     *(addr_in_vm - 1) = DEFAULT_COLOUR;
     ^~~~~~~~~~~~~~~~~
main.c:1282:5: error: invalid type argument of unary '*' (have 'unsigned int')
     *(addr_in_vm + 1) = key;
     ^~~~~~~~~~~~~~~~~
main.c:1283:5: error: invalid type argument of unary '*' (have 'unsigned int')
     *(addr_in_vm + 2) = DEFAULT_COLOUR;
     ^~~~~~~~~~~~~~~~~
main.c: In function 'tty_do_write':
main.c:1318:20: error: 'CONSOLE' {aka 'struct <anonymous>'} has no member named 'counter'
  while(tty->console->counter > 0){
                    ^~
main.c:1319:37: error: 'CONSOLE' {aka 'struct <anonymous>'} has no member named 'tail'
   unsigned char key = *(tty->console->tail);
                                     ^~
main.c:1320:15: error: 'CONSOLE' {aka 'struct <anonymous>'} has no member named 'tail'
   tty->console->tail++;
               ^~
main.c:1321:15: error: 'CONSOLE' {aka 'struct <anonymous>'} has no member named 'counter'
   tty->console->counter--;
               ^~
main.c:1322:18: error: 'CONSOLE' {aka 'struct <anonymous>'} has no member named 'tail'
   if(tty->console->tail == tty->buf + KEYBOARD_BUF_SIZE){
                  ^~
main.c:1323:16: error: 'CONSOLE' {aka 'struct <anonymous>'} has no member named 'tail'
    tty->console->tail == tty->buf;
                ^~
main.c: In function 'init_tty':
main.c:1356:25: warning: assignment to 'unsigned int *' from incompatible pointer type 'unsigned char *' [-Wincompatible-pointer-types]
   tty->head = tty->tail = tty->buf;
                         ^
make: *** [Makefile:32: kernel.bin] Error 1
```



```shell
unsigned int addr_in_vm = VM_BASE_ADDR + tty->console->cursor * 2;

main.c:1270:5: error: invalid type argument of unary '*' (have 'unsigned int')
     *(addr_in_vm - 2) = ' ';
     ^~~~~~~~~~~~~~~~~
```

## 总结

我简化了终端切换的按键，只需点击“F1~F3"中的一个键就能在console0、console1、console2三个终端之间切换。

TTY的任务，耗费了五天时间。

为什么这么慢？

因为我又变得非常懒惰。每天的工作时间不足8小时。有很多天，我只在上午学习，午休之后就不想学。

还有一个原因，回顾知识用得时间太长。一定要记住，实在想不起来，就去看学习资料。