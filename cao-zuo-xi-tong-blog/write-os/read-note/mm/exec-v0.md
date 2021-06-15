# 怎么实现exec

这是个难点。





## ELF

ELF header的定义可以在 `/usr/include/elf.h` 中找到。`Elf32_Ehdr`是32位 ELF header的结构体。`Elf64_Ehdr`是64位ELF header的结构体。

`Elf32_Ehdr`是操作系统内置的结构体，具体内容是：

```c
typedef struct
{
  unsigned char e_ident[EI_NIDENT];     /* Magic number and other info */
  Elf32_Half    e_type;                 /* Object file type */
  Elf32_Half    e_machine;              /* Architecture */
  Elf32_Word    e_version;              /* Object file version */
  Elf32_Addr    e_entry;                /* Entry point virtual address */
  Elf32_Off     e_phoff;                /* Program header table file offset */
  Elf32_Off     e_shoff;                /* Section header table file offset */
  Elf32_Word    e_flags;                /* Processor-specific flags */
  Elf32_Half    e_ehsize;               /* ELF header size in bytes */
  Elf32_Half    e_phentsize;            /* Program header table entry size */
  Elf32_Half    e_phnum;                /* Program header table entry count */
  Elf32_Half    e_shentsize;            /* Section header table entry size */
  Elf32_Half    e_shnum;                /* Section header table entry count */
  Elf32_Half    e_shstrndx;             /* Section header string table index */
} Elf32_Ehdr;
```



```c
typedef struct
{
  Elf32_Word    p_type;                 /* Segment type */
  Elf32_Off     p_offset;               /* Segment file offset */
  Elf32_Addr    p_vaddr;                /* Segment virtual address */
  Elf32_Addr    p_paddr;                /* Segment physical address */
  Elf32_Word    p_filesz;               /* Segment size in file */
  Elf32_Word    p_memsz;                /* Segment size in memory */
  Elf32_Word    p_flags;                /* Segment flags */
  Elf32_Word    p_align;                /* Segment alignment */
} Elf32_Phdr;
```



```c
/* Legal values for p_type (segment type).  */

#define PT_NULL         0               /* Program header table entry unused */
#define PT_LOAD         1               /* Loadable program segment */
#define PT_DYNAMIC      2               /* Dynamic linking information */
#define PT_INTERP       3               /* Program interpreter */
#define PT_NOTE         4               /* Auxiliary information */
#define PT_SHLIB        5               /* Reserved */
#define PT_PHDR         6               /* Entry for header table itself */
#define PT_TLS          7               /* Thread-local storage segment */
#define PT_NUM          8               /* Number of defined types */
#define PT_LOOS         0x60000000      /* Start of OS-specific */
#define PT_GNU_EH_FRAME 0x6474e550      /* GCC .eh_frame_hdr segment */
#define PT_GNU_STACK    0x6474e551      /* Indicates stack executability */
#define PT_GNU_RELRO    0x6474e552      /* Read-only after relocation */
#define PT_LOSUNW       0x6ffffffa
#define PT_SUNWBSS      0x6ffffffa      /* Sun Specific segment */
#define PT_SUNWSTACK    0x6ffffffb      /* Stack segment */
#define PT_HISUNW       0x6fffffff
#define PT_HIOS         0x6fffffff      /* End of OS-specific */
#define PT_LOPROC       0x70000000      /* Start of processor-specific */
#define PT_HIPROC       0x7fffffff      /* End of processor-specific */
```

## 小结

### 2021-06-11 12:25

逐行看do_exec的实现。很难理解，仍没有理解。耗时50 + 7分。

do_exec，用其他指令替换当前新建的进程的指令。

为什么慢？

1. 这个函数的代码混入ELF知识。我看到这东西就有点怕，复杂，我又没有记住它的结构。
2. 不理解，有很多不影响写代码的问题。
   1. 用exec要执行的命令的指令替换子进程的指令，替换的是LDT吗？还是替换的其他啥东西？
      1. 进程的进程表存储在哪里？存储进程表的内存空间，不属于进程的内存空间。
      2. 进程的指令，在进程的内存空间中；进程的数据，也在进程的内存空间中。指令和数据，都在LDT描述的内存空间中。
      3. 进程的堆栈，在哪里？在LDT描述的内存空间中吗？

### 2021-06-11 15:55

逐行看do_exec的实现。很难理解，仍没有理解。耗时1个小时49分 + 1个小时。

没有特效方法，看不懂代码的时候，我盯着代码看。有时突然就理解了一点。

有疑问的代码是：

```c
PUBLIC int do_exec()
{
	// some code

	/* setup the arg stack */
	int orig_stack_len = mm_msg.BUF_LEN;
	char stackcopy[PROC_ORIGIN_STACK];
	phys_copy((void*)va2la(TASK_MM, stackcopy),
		  (void*)va2la(src, mm_msg.BUF),
		  orig_stack_len);

	u8 * orig_stack = (u8*)(PROC_IMAGE_SIZE_DEFAULT - PROC_ORIGIN_STACK);

	int delta = (int)orig_stack - (int)mm_msg.BUF;

	// 不懂这段代码存在的必要性。
  // 去掉这段后，不能运行。
  int argc = 0;
	if (orig_stack_len) {	/* has args */
		char **q = (char**)stackcopy;
		for (; *q != 0; q++,argc++)
			*q += delta;
	}

	phys_copy((void*)va2la(src, orig_stack),
		  (void*)va2la(TASK_MM, stackcopy),
		  orig_stack_len);
  
	// some code
}
```



理解过程如下：

1. `u8 * orig_stack = (u8*)(PROC_IMAGE_SIZE_DEFAULT - PROC_ORIGIN_STACK);`
   1. 盯着看了很久，突然联想到内存地址，就理解了。
   2. orig_stack 是栈的最低地址在进程的内存空间中的偏移量。偏移量是在某种意义（哪种意义？物理地址？相对地址？线性地址？虚拟地址？我不记得）就是内存地址。
   3. orig_stack是栈的最低地址在进程的内存空间中的内存地址。它就是内存地址。
2. `int delta = (int)orig_stack - (int)mm_msg.BUF;`是栈的最低地址在不同进程的内存中的内存地址的差值。



```c
int argc = 0;
	if (orig_stack_len) {	/* has args */
		char **q = (char**)stackcopy;
		for (; *q != 0; q++,argc++)
			*q += delta;
	}
```



栈的最低地址修改了，栈中每个元素的地址都需要修改。



stackcopy 中存储的究竟是什么？断点瞧瞧吧。断点观察数据，我也没有看明白是怎么回事。



![image-20210612181032603](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/mm/image-20210612181032603.png)



`char **q = (char**)stackcopy;`的汇编代码：

```shell
8346     7469:       8d 85 34 fb ff ff       lea    -0x4cc(%ebp),%eax
8347     746f:       89 45 ec                mov    %eax,-0x14(%ebp)
8348     7472:       eb 17                   jmp    748b <do_exec+0x290>
```



耗时5个小时，理解不了修改stackcopy的这段代码存在的必要性。

又耗费了3个小时38分钟，仍然理解不了。

我做了什么？

1. 把这个循环抽出来写成单独的代码。失败了。不能访问某个内存。
2. 在汇编中断点调试，什么也没有验证出来。
3. 又回到C代码中断点调试，疑问更多了，仍然理解不了。

不知道怎么思考，无逻辑思考，思维散漫、发散，期待灵感降临。

大部分时间耗费在断点调试和无根据的猜想。非常非常低效。沮丧，无用。



![image-20210613004358268](/Users/cg/Documents/gitbook/my-note-book/cao-zuo-xi-tong-blog/write-os/read-note/mm/image-20210613004358268.png)



xp /1wx 0xd:(0x000283fc-1232)

xp /1bc 0xd:(0x000283fc-1232)

xp /1bc (0x00027f2c+12)



xp /1wx 0xd:(0x283fc-1236)

xp /1gc 0xd:(0x283fc-1236)

xp /1wc 0xd:(0x000283fc-1236)



### 2021-06-13 11:12

理解了上面那个小节的问题。具体说，理解了下面的代码：

```c
PUBLIC int do_exec()
{
	// some code

	/* setup the arg stack */
	int orig_stack_len = mm_msg.BUF_LEN;
	char stackcopy[PROC_ORIGIN_STACK];
	phys_copy((void*)va2la(TASK_MM, stackcopy),
		  (void*)va2la(src, mm_msg.BUF),
		  orig_stack_len);

	u8 * orig_stack = (u8*)(PROC_IMAGE_SIZE_DEFAULT - PROC_ORIGIN_STACK);

	int delta = (int)orig_stack - (int)mm_msg.BUF;

	// 不懂这段代码存在的必要性。
  // 去掉这段后，不能运行。
  int argc = 0;
	if (orig_stack_len) {	/* has args */
		char **q = (char**)stackcopy;
		for (; *q != 0; q++,argc++)
			*q += delta;
	}

	phys_copy((void*)va2la(src, orig_stack),
		  (void*)va2la(TASK_MM, stackcopy),
		  orig_stack_len);
  
	// some code
}
```



为什么能够理解？我看了其他函数，知道了这个函数中的mm_msg.BUF的结构。

在哪里能看到mm_msg.BUF的结构？

```c
// /Users/cg/data/code/os/yy-os/osfs10/e/lib/exec.c
PUBLIC int execv(const char *path, char * argv[])
{
	char **p = argv;
	char arg_stack[PROC_ORIGIN_STACK];
	int stack_len = 0;

  // 统计参数的个数，因为参数中有一个是0，所以，可以用*p作为循环条件。
	while(*p++) {
		assert(stack_len + 2 * sizeof(char*) < PROC_ORIGIN_STACK);
		stack_len += sizeof(char*);
	}

	*((int*)(&arg_stack[stack_len])) = 0;
	stack_len += sizeof(char*);

  // 1. 在arg_stack中存储了两部分数据。前面存储指针地址，后面存储参数。
  // 2. 从stack_len开始存储参数，前面存储的是后面参数的内存地址。
  // 3. 知道arg_stack是这个结构非常重要。不知道这个结构，就理解不了dO_exec的汇编代码实现的功能，还会和我和我掌握C语言
  //    指针知识矛盾。
	char ** q = (char**)arg_stack;
	for (p = argv; *p != 0; p++) {
    // 在arg_stack的前面记录后面存储的参数的内存地址。
		*q++ = &arg_stack[stack_len];

		assert(stack_len + strlen(*p) + 1 < PROC_ORIGIN_STACK);
		strcpy(&arg_stack[stack_len], *p);
		stack_len += strlen(*p);
		arg_stack[stack_len] = 0;
		stack_len++;
	}

	MESSAGE msg;
	msg.type	= EXEC;
	msg.PATHNAME	= (void*)path;
	msg.NAME_LEN	= strlen(path);
	msg.BUF		= (void*)arg_stack;
	msg.BUF_LEN	= stack_len;

	send_recv(BOTH, TASK_MM, &msg);
	assert(msg.type == SYSCALL_RET);

	return msg.RETVAL;
}
```



万分沮丧之时，还是找到了解决问题的方法。不是依靠别人，也不是依靠灵感，而是靠我继续看代码，具体说，依靠我联系相关代码继续研究这个问题。

山重水复疑无路，柳岸花明又一村。又一次体会了这种感觉。所以，不要放弃，我一定能克服还会遇到的各种困难，写完这个简单的操作系统。

经验是：联系起来看代码。

另外，遇到问题，不要分神，不要心算，要思路清晰地推理。

我试图用各种没有理论依据的想法去牵强地理解这段修改栈的代码，不成功。这种方法很危险。假如，我在精神不好时误把错误当正确，岂不是跳过了问题。

依据经验理解不了C代码，就看C代码对应的汇编代码吧。看了汇编代码，就能准确理解那些C代码在做什么。

还有几个遗留问题：

```c
/* setup the arg stack */
	int orig_stack_len = mm_msg.BUF_LEN;
	char stackcopy[PROC_ORIGIN_STACK];
	phys_copy((void*)va2la(TASK_MM, stackcopy),
		  (void*)va2la(src, mm_msg.BUF),
		  orig_stack_len);

	u8 * orig_stack = (u8*)(PROC_IMAGE_SIZE_DEFAULT - PROC_ORIGIN_STACK);

	int delta = (int)orig_stack - (int)mm_msg.BUF;

	int argc = 0;
	if (orig_stack_len) {	/* has args */
    // 为什么需要把mm_msg.BUF复制到本进程再处理？在下面又把处理过的栈空间返回给源进程，为何不直接在本进程
    // 修改源进程的栈？
    // 在一个进程中修改另外一个进程，似乎不是通常操作。
    // 不过，本函数通常运行在内核态吗？
		char **q = (char**)stackcopy;
		for (; *q != 0; q++,argc++)
			*q += delta;
	}

	phys_copy((void*)va2la(src, orig_stack),
		  (void*)va2la(TASK_MM, stackcopy),
		  orig_stack_len);

	proc_table[src].regs.ecx = argc; /* argc */
	proc_table[src].regs.eax = (u32)orig_stack; /* argv */

	/* setup eip & esp */
	proc_table[src].regs.eip = elf_hdr->e_entry; /* @see _start.asm */
	proc_table[src].regs.esp = PROC_IMAGE_SIZE_DEFAULT - PROC_ORIGIN_STACK;

	strcpy(proc_table[src].name, pathname);
```

1. 栈为什么需要重新放置？
2. 看代码中的注释。
3. 修改了进程映像的栈、寄存器，就能改变这个进程要执行的指令。总感觉哪里理解不了。不过，可以记住。下次要做这种事，我也可以这么做。
4. 执行execv的子进程为啥不需要执行exit。
   1. 经验证，是否执行exit都不会影响执行效果。
   2. 我不理解。不执行exit，应该会异常啊。暂时想不明白。

耗时2个小时。成果突出，我比较满意。

### 2021-06-13 14:42



### 2021-06-14 14:12

用gdb断点查看数据来验证我对do_exec中修改栈的代码的理解。失败了。耗时1小时16分。

用gdb查看到的内存地址、变量，都不是在我的操作系统中的内存地址和变量值。

为什么这么说？

在

```c
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



`int fd = open(argv[0], O_RDWR);`

1. argv[0]不是我在终端输入的任何字母。
2. fd 的值是1040911104。在我的操作系统中，这是绝对不可能的。因为，fd的最大值只能是63。
3. 在do_open中，我终于看到了，接收到的argv[0]经过处理后，变成了我在终端输入的字符串，例如“echo"。

看到了这些，我决定不再用gdb或bochs断点查看数据来验证修改栈的代码。

在我没有弄清楚gdb和我的操作系统中的变量、内存地址的关系之前，我的想法行不通。

我主要只能依靠观察执行效果来验证我的代码是否正确。

先不管这个问题了。遇到实在绕不过这个问题的时候再研究它。

我大部分时间在干什么？

1. 在不同的地方设置断点，观察数据。
2. 数据不正确，我举不同例子观察数据。
3. 反复重试以上步骤。
4. 在大概三五分钟内，我发现了明显的数据异常，做出了结论：GDB观察到的并非我的操作系统中的内存地址和数据。

### 2021-06-14 17:35

理解execv（包括do_exec）等。对下面的问题做出了合理猜想：

1. 为什么需要重新放置进程的栈？
2. 进程的栈在进程的内存中的什么位置？

耗时1个小时9分钟。

我在做什么？看代码，合理猜想，没有执行代码，没有断点查看数据。毫无章法地猜想。是否正确，我不知道，无从验证。

x 是 examine 的缩写，意思是检查。

n表示要显示的内存单元的个数，比如：20

f表示显示方式, 可取如下值：

x 按十六进制格式显示变量。
d 按十进制格式显示变量。
u 按十进制格式显示无符号整型。
o 按八进制格式显示变量。
t 按二进制格式显示变量。
a 按十六进制格式显示变量。
i 指令地址格式
c 按字符格式显示变量。
f 按浮点数格式显示变量。
————————————————
**u表示一个地址单元的长度：**

```
b表示单字节，
h表示双字节，
w表示四字节，
g表示八字节
1234
```

## 测试

```
x /20xh 0x7fffffffe080
```

