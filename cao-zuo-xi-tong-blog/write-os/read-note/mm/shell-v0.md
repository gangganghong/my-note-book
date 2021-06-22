# 怎么实现一个简单的shell

## v0

> 我自己的回忆，正确性很差。

### 流程

怎么实现简单的shell？

1. 在Init中调用fork，创建子进程。
2. 在子进程中调用shabby_shell。
3. shabby_shell做什么？
4. 在shabby_shell中再次调用fork，创建子进程。
5. 在子进程中调用execv。
6. execv的参数，从终端获取。
7. 如果从终端获取的输入数据，解析出来后，
   1. 第一个数据是要执行的命令。它是一个可执行的二进制文件。
   2. 如果这个二进制文件存在于硬盘中，执行它。
   3. 如果这个二进制文件不存在，原样输出这个命令名称。

### 疑问

1. shell怎么和终端联系起来？

什么叫联系起来？从运行效果看，shell获取输入数据、向用户展示输出数据都需要通过终端。

输入、输出数据分别使用read、write。这两个函数都需要使用open打开一个文件。把open的那个文件设置成终端名，就能在当前shell使用终端名对应的那个终端。

直接回答疑问：shell通过文件名和终端联系起来的。

耗时1个小时07分，没有看明白shell和终端是怎么联系起来的，不明白文件名如何在其中起作用。

```shell
void shabby_shell(const char * tty_name)
{
	int fd_stdin  = open(tty_name, O_RDWR);
	assert(fd_stdin  == 0);
	int fd_stdout = open(tty_name, O_RDWR);
	assert(fd_stdout == 1);

	char rdbuf[128];

	while (1) {
		write(1, "$ ", 2);
		int r = read(0, rdbuf, 70);
		rdbuf[r] = 0;

		int argc = 0;
		char * argv[PROC_ORIGIN_STACK];
		char * p = rdbuf;
		char * s;
		int word = 0;
		char ch;
		do {
			ch = *p;
			if (*p != ' ' && *p != 0 && !word) {
				s = p;
				word = 1;
			}
			if ((*p == ' ' || *p == 0) && word) {
				word = 0;
				argv[argc++] = s;
				*p = 0;
			}
			p++;
		} while(ch);
		argv[argc] = 0;

		int fd = open(argv[0], O_RDWR);
		if (fd == -1) {
			if (rdbuf[0]) {
				write(1, "{", 1);
				write(1, rdbuf, r);
				write(1, "}\n", 2);
			}
		}
		else {
			close(fd);
			int pid = fork();
			if (pid != 0) { /* parent */
				int s;
				wait(&s);
			}
			else {	/* child */
				execv(argv[0], argv);
			}
		}
	}

	close(1);
	close(0);
}
```

更准确地说，我不明白

```c
int fd_stdin  = open(tty_name, O_RDWR);
	assert(fd_stdin  == 0);
	int fd_stdout = open(tty_name, O_RDWR);
	assert(fd_stdout == 1);
```

为何需要在shell中重新打开终端文件而不能沿用Init中打开的终端文件？

沿用Init中打开的终端文件时，切换到tty1和tty2后，不能输入字符；在tty0能输入字符。

我不明白原因。

## v2

```c
// 不管是int *还是char *，占用都是4个字节。
*((int*)(&arg_stack[stack_len])) = 0;
```

对应的汇编代码是

![image-20210615230821815](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/mm/image-20210615230821815.png)

从汇编代码看出，只是在arg_stack的某个位置的内存中放置了0x0。不能看出0x0占用了几个字节。0x0是一个int类型，存储它当然需要使用4个字节。

对这个指针语句，使用我摸索出的指针经验也行，用汇编来理解也行。

用汇编来理解，这条语句做的事是：

1. 找到arg_stack的内存地址D。
2. 找到目标内存地址D+A。
3. 从内存地址为(D+A)的内存空间开始，使用4个字节存储0x0。

## v3

到处是疑问。沮丧。

Init进程为何不把大部分进程体放在循环中？

当然，我现在可以代入具体例子来理解，这样是合理的。若我自己写代码，我是难以写成这样的。

面对这种问题，我回答不了。

## 小结

### 2021-06-15 22:08

耗时1个小时07分，没有看明白shell和终端是怎么联系起来的，不明白文件名如何在其中起作用。

我在各种代码中看来看去，文件系统，不得不说，有点复杂。这是IPC导致的。

我决定，搁置这个问题，在看文件系统时再研究。

### 2021-06-15 23:28

阅读execv。耗时1小时18分。

做了两件事：

1. 解析"echo hello world"。我自己想出算法比较费劲。
   1. 遇到非零非空字符并且还没有记录字符地址时时，记录该字符的地址，
   2. 遇到空字符或零字符，并且记录了某个字符的地址时，遍历一个单词结束。
2. 指针语句：`*((int*)(&arg_stack[stack_len])) = 0;`。

还有那么多内容，何时是个头啊？

### 2021-06-16 06:22

困惑于shell的进程设计。耗时。

我只能记忆，不能理解，自己设计不出这种结构。