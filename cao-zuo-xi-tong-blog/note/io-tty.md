# 输入/输出系统---tty任务

## 零散笔记

### 框架

#### 伪代码

```c

while(1){
			// 读数据
  		tty_do_read();
  		// 写数据
  		tty_do_write();
}


tty_do_read(){
  	// 如果是当前tty
  	if(is_current_console){
      		// 从键盘读取数据
    }
}

tty_do_write(){
  		// 向对应tty的显存区域写入数据
}
```

#### 疑问

向显存写数据的时候为啥不需分辨是否为当前tty？看不出问题。

模拟运行：

1. 当前是tty1。
2. 敲击键盘，将数据存储到tty1的缓冲区。
3. 遍历到tty1，tty1的缓冲区由数据，读取并写入到tty1对应的console1中。



时钟中断中断例程处理进程调度，键盘中断中断例程处理键盘输入。

遍历时，遇到当前tty才读写。如此，没有看到效果有啥问题。

在这种小细节上耗费时间，不应该。

### 全局视角

tty进程通过`restart`运行起来，`keyboard_handler`通过键盘中断运行起来。

tty进程运行起来后，在时钟中断例程调度程序时进入”休眠“状态。

敲击键盘，键盘中断例程把数据通过`keyboard_handler`放进键盘输入缓冲区。

tty进程”休眠“后，在哪里再次获得CPU控制权？

我无法模拟这种多进程的运行情况。

我的猜测是：tty进程”休眠“后，`TestA`等用户进程轮流运行，最终一定会全部运行完，恢复tty进程；时钟中断再次发生，tty进程又”休眠“。

不猜了，毫无意义。或许，这些进程根本就不是我猜想的那样运行。

找时间运行代码测试。

知道了实际运行结果有用吗？下一次再遇到这种多进程，我能模拟出正确的执行过程吗？若不能，那我测试它干嘛。

注意，有两个缓存区。

第一个，敲击键盘，数据会被放入一个缓冲区C1。

第二个，tty进程会把C1中的数据放到缓冲区C2。

C1和C2的结构类似，伪代码大概是下面这样的。

```c
typedef struct{
  	// 下一个要处理的数据的位置
  	int		head;
  	// 下一个空闲位置
  	int 	tail;
  	// 当前数据的数量，字节数
  	int		count;
  	// 存储数据的缓冲区
  	int		buf[256];
}Cache;
```

### console

```c
// global.c
PUBLIC	CONSOLE		console_table[NR_CONSOLES];
```

有这么一句。我先入为主，认为，必须在某个地方初始化了`console_table`，在函数`init_screen`中才能使用这个数组。殊不知，初始化`console_table`就在`init_screen`中。

```c
// /Users/cg/data/code/os/yy-os/osfs07/k/kernel/console.c
/*======================================================================*
			   init_screen
 *======================================================================*/
PUBLIC void init_screen(TTY* p_tty)
{
	// 这个用法挺奇特的，我不理解。两个
  int nr_tty = p_tty - tty_table;
	p_tty->p_console = console_table + nr_tty;

	int v_mem_size = V_MEM_SIZE >> 1;	/* 显存总大小 (in WORD) */

	int con_v_mem_size                   = v_mem_size / NR_CONSOLES;
	p_tty->p_console->original_addr      = nr_tty * con_v_mem_size;
	p_tty->p_console->v_mem_limit        = con_v_mem_size;
	p_tty->p_console->current_start_addr = p_tty->p_console->original_addr;

	/* 默认光标位置在最开始处 */
	p_tty->p_console->cursor = p_tty->p_console->original_addr;

	if (nr_tty == 0) {
		/* 第一个控制台沿用原来的光标位置 */
		p_tty->p_console->cursor = disp_pos / 2;
		disp_pos = 0;
	}
	else {
		out_char(p_tty->p_console, nr_tty + '0');
		out_char(p_tty->p_console, '#');
	}

	set_cursor(p_tty->p_console->cursor);
}
```

C语言中，获取C的数组元素的索引的方法，在下面代码中能找到。先声明数组，然后再遍历赋值。`console_table`就是这样处理的。

```c
#include <stdio.h>

int main(int argc, char **argv)
{
        int arr[5];

        for(int i = 0; i < 5; i++){
                arr[i] = i;
        }

        int *ptr = arr;

        ptr++;
        printf("*ptr - arr = %d\n", ptr - arr);

        ptr++;
        printf("*ptr - arr = %d\n", ptr - arr);

        ptr++;
        printf("*ptr - arr = %d\n", ptr - arr);

        for(int j = 0; j < 5; j++){
                printf("arr[%d] = %d\n", j, j);
        }

        return 0;
}
```

执行结果是：

```shell
[root@localhost c]# gcc -o arr arr.c
[root@localhost c]# ./arr
*ptr - arr = 1
*ptr - arr = 2
*ptr - arr = 3
arr[0] = 0
arr[1] = 1
arr[2] = 2
arr[3] = 3
arr[4] = 4
```

`ptr - arr`是两个内存地址相减吗？结果不应该是8个字节吗？因为`arr`的每个元素占用8个字节。

理解不了。只能暂时当作语法记住了。

另外，用这个命令编译`gcc -o -g arr arr.c`会报错：

```shell
[root@localhost c]# gcc -o -g arr arr.c
arr: In function `_fini':
(.fini+0x0): multiple definition of `_fini'
/usr/lib/gcc/x86_64-redhat-linux/8/../../../../lib64/crti.o:(.fini+0x0): first defined here
arr: In function `data_start':
(.data+0x0): multiple definition of `__data_start'
/usr/lib/gcc/x86_64-redhat-linux/8/../../../../lib64/crt1.o:(.data+0x0): first defined here
arr:(.rodata+0x8): multiple definition of `__dso_handle'
/usr/lib/gcc/x86_64-redhat-linux/8/crtbegin.o:(.rodata+0x0): first defined here
arr:(.rodata+0x0): multiple definition of `_IO_stdin_used'
/usr/lib/gcc/x86_64-redhat-linux/8/../../../../lib64/crt1.o:(.rodata.cst4+0x0): first defined here
arr: In function `_dl_relocate_static_pie':
(.text+0x30): multiple definition of `_dl_relocate_static_pie'
/usr/lib/gcc/x86_64-redhat-linux/8/../../../../lib64/crt1.o:(.text+0x30): first defined here
arr: In function `_start':
(.text+0x0): multiple definition of `_start'
/usr/lib/gcc/x86_64-redhat-linux/8/../../../../lib64/crt1.o:(.text+0x0): first defined here
arr: In function `_init':
(.init+0x0): multiple definition of `_init'
/usr/lib/gcc/x86_64-redhat-linux/8/../../../../lib64/crti.o:(.init+0x0): first defined here
/tmp/ccDrXx5O.o: In function `main':
arr.c:(.text+0x0): multiple definition of `main'
arr:(.text+0xe6): first defined here
/usr/lib/gcc/x86_64-redhat-linux/8/crtend.o:(.tm_clone_table+0x0): multiple definition of `__TMC_END__'
arr:(.data+0x8): first defined here
/usr/bin/ld: error in arr(.eh_frame); no .eh_frame_hdr table will be created.
collect2: error: ld returned 1 exit status
```

在哪里写入数据到显存？就在这里。

```c
// /Users/cg/data/code/os/yy-os/osfs07/j/kernel/console.c
/*======================================================================*
			   out_char
 *======================================================================*/
PUBLIC void out_char(CONSOLE* p_con, char ch)
{
	u8* p_vmem = (u8*)(V_MEM_BASE + p_con->cursor * 2);

	*p_vmem++ = ch;
	*p_vmem++ = DEFAULT_CHAR_COLOR;
	p_con->cursor++;

	set_cursor(p_con->cursor);
}
```

`u8* p_vmem = (u8*)(V_MEM_BASE + p_con->cursor * 2);`，将指针指向显存的某个位置，然后通过移动指针修改某个位置的数据，例如`*p_vmem++ = ch;`。

### 大疑问

tty进程休眠后，调度程序理论上一直不分配CPU时间给它，tty又是怎么获得CPU控制权的呢？

1. tty和其他普通进程一起初始化，其他进程是`TestA`等。
2. 在时钟中断的进程调度中，tty进程不会被分配CPU运行时间。
3. 敲击键盘，8048监测到键盘操作，会读取数据并且把数据发送给8042，8042告知8259A发生了键盘中断。
4. 在键盘中断例程中，会把数据从8042的缓冲区读取到tty的缓冲区。
5. 再从tty的缓冲区把数据读取出来写入到显存。
6. 写到了显存，我们就能在屏幕上看到打印出来的字符。

### tty任务框架

tty结构

```c
typedef	struct {
  	// 显存初始位置
  	int original_address;
  	// 当前终端的显存大小
  	int	limit;
  	// 光标所在的显存位置。
  	int	current_address;
}CONSOLE;

typedef struct {
  		// tty的缓冲区
  		char	kb_in_buff[256];
  		int	head;
  		int tail;
  		int	count;
  		// 本tty对应的console
  		CONSOLE *console;
}TTY;
```

上面的tty结构完全是因为看书中的代码后复述出来的，我并不理解如此设计的必要性。



## Mac 操作

切换tty使用快捷键`fn + f1`等，不是`Alt + fn`。

先按`fn`，等键盘上的电子bar出现`F1~~F12`后，再按`fn + f1`或`fn + f2`或`fn + f3`切换tty。