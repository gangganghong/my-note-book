---
description: 在mac上验证过的命令
---

# 常用linux命令

### 用grep找出并杀死进程

```text
# grep -v 'grep' 排除包含grep自身的进程
ps -ef | grep qemu  | grep -v 'grep' | awk '{print $2}' | xargs kill -9
```

### 字符串替换sed

```shell
sed -i 's/\/home\/cg\/tools\/bochs-2.6.11\/share\/bochs/\/usr\/local\/share\/bochs/g' code.c
```

将 code.c 中的 `\/home\/cg\/tools\/bochs-2.6.11\/share\/bochs` 全部替换成 `\/usr\/local\/share\/bochs` 。`\/` 是对 `/` 进行转义。

### 安装bochs

```shell
/configure --prefix=/home/cg/tools/bochs-2.6.11 --enable-plugins   --enable-x86-64   --enable-cpp   --enable-debugger   --enable-disasm   --enable-gdb-stub --enable-x86-debugger   --enable-e1000 
# 报错
configure: error: --enable-debugger and --enable-gdb-stub are mutually exclusive  
# 解决：--enable-debugger and --enable-gdb-stub  不能共存。--enable-gdb-stub 能使用gdb调试C代码。
./configure --prefix=/home/cg/tools/bochs-2.6.11 --enable-plugins   --enable-x86-64   --enable-cpp  --enable-disasm   --enable-gdb-stub --enable-x86-debugger --enable-e1000 
# 报错
make: *** No rule to make target 'misc/bximage.cc', needed by 'misc/bximage.o'.  Stop.
# 解决：修改bochs源码中所有Makefile.in中的所有后缀.cc改成.cpp；编译时加上--enable-cpp=yes
./configure --prefix=/home/cg/tools/bochs-2.6.11   --enable-gdb-stub --enable-cpp=yes
```



### 查看linux最大进程数

```shell
# 查看linux最大进程数
[root@localhost e]# cat /proc/sys/kernel/pid_max
4194304
# 限制用户进程
# 查看文档
[root@localhost e]# help ulimit
ulimit: ulimit [-SHabcdefiklmnpqrstuvxPT] [limit]
    Modify shell resource limits.

    Provides control over the resources available to the shell and processes
    it creates, on systems that allow such control.

    Options:
      -S	use the `soft' resource limit
      -H	use the `hard' resource limit
      -a	all current limits are reported
      -b	the socket buffer size
      -c	the maximum size of core files created
      -d	the maximum size of a process's data segment
      -e	the maximum scheduling priority (`nice')
      -f	the maximum size of files written by the shell and its children
      -i	the maximum number of pending signals
      -k	the maximum number of kqueues allocated for this process
      -l	the maximum size a process may lock into memory
      -m	the maximum resident set size
      -n	the maximum number of open file descriptors
      -p	the pipe buffer size
      -q	the maximum number of bytes in POSIX message queues
      -r	the maximum real-time scheduling priority
      -s	the maximum stack size
      -t	the maximum amount of cpu time in seconds
      -u	the maximum number of user processes
      -v	the size of virtual memory
      -x	the maximum number of file locks
      -P	the maximum number of pseudoterminals
      -T	the maximum number of threads

    Not all options are available on all platforms.

    If LIMIT is given, it is the new value of the specified resource; the
    special LIMIT values `soft', `hard', and `unlimited' stand for the
    current soft limit, the current hard limit, and no limit, respectively.
    Otherwise, the current value of the specified resource is printed.  If
    no option is given, then -f is assumed.

    Values are in 1024-byte increments, except for -t, which is in seconds,
    -p, which is in increments of 512 bytes, and -u, which is an unscaled
    number of processes.

    Exit Status:
    Returns success unless an invalid option is supplied or an error occurs.
```



### 文件解压

```shell
tar -zxvf java.tar.gz
gzip -d java.gz
```

```shell
[root@localhost d]# tar cf cm.tar command/*
[root@localhost d]# ls
80m.img   a.img    bochsrc-gdb  boot    command  include  kernel.bin  lib      m-untar.c  scripts
Makefile  bochsrc  bochsrc-old  cm.tar  fs       kernel   krnl.map    m-untar  mm         xxd.sh
[root@localhost d]# tar tf cm.tar
command/Makefile
command/command.tar
command/echo.c
command/pwd.c
command/start.asm
```



## shell

```shell
#! /bin/bash
echo $0
#echo $1
#echo $2
arg1=$1
arg2=$2
#if [ ! $arg1  || ! $arg2 ]
if [[ ! $arg1  && ! $arg2 ]]
then
        # xxd -u -a -g 1 -c 16 -s $arg1 -l 512 $arg2
        echo "usage: ./xxd.sh 跳过的字节 文件名"
else
        # echo "usage: ./xxd.sh 跳过的字节 文件名"
        xxd -u -a -g 1 -c 16 -s $arg1 -l 512 $arg2
fi
```

1. If...else结构。
2. 逻辑运算，需要使用 [[ ]] 。
3. 判断变量是否为空，使用 ! $var 。
4. 接收参数。$0 是当前执行的shell文件名。

执行

```shell
[root@localhost c]# ./xxd.sh 0xA01800 80m.img
./xxd.sh
00a01800: 01 00 00 00 2E 00 00 00 00 00 00 00 00 00 00 00  ................
00a01810: 02 00 00 00 64 65 76 5F 74 74 79 30 00 00 00 00  ....dev_tty0....
00a01820: 03 00 00 00 64 65 76 5F 74 74 79 31 00 00 00 00  ....dev_tty1....
00a01830: 04 00 00 00 64 65 76 5F 74 74 79 32 00 00 00 00  ....dev_tty2....
00a01840: 05 00 00 00 63 6D 64 2E 74 61 72 00 00 00 00 00  ....cmd.tar.....
00a01850: 06 00 00 00 6B 65 72 6E 65 6C 2E 62 69 6E 00 00  ....kernel.bin..
00a01860: 07 00 00 00 65 63 68 6F 00 00 00 00 00 00 00 00  ....echo........
00a01870: 08 00 00 00 70 77 64 00 00 00 00 00 00 00 00 00  ....pwd.........
00a01880: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
*
00a019f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
```



00000243403

258352

83715

## rm

```shell
# 删除 cm.tar m-untar 外的其他文件
ls  | egrep -v '(cm.tar|m-untar*)' | xargs rm -rvf
ls  | grep -v cm.tar | xargs rm -rvf
```



## dd 和 seek

假如我有一个文件abc.gz，大小为83456k，我想用dd命令实现如下备份 结果：首先将备份分成三个部分，第一部分为备份文件abc.gz的前10000k，第二部分为中间的70000k，最后备份后面的3456k. 

备份方法如下三条命令： 
dd if=abc.gz of=abc.gz.bak1 bs=1k count=10000
dd if=abc.gz of=abc.gz.bak2 bs=1k skip=10000 count=70000 
dd if=abc.gz of=abc.gz.bak3 bs=1k skip=80000 

恢复方法如下：
dd if=abc.gz.bak1 of=abc.gz
dd if=abc.gz.bak2 of=abc.gz bs=1k seek=10000
dd if=abc.gz.bak3 of=abc.gz bs=1k seek=80000

这时你查看一下恢复的文件将和你原来的文件一模一样，说明备份成功!

理解说明:skip=xxx是在备份时对if 后面的部分也就是原文件跳过多少块再开始备份;seek=xxx则是在备份时对of 后面的部分也就是目标文件跳过多少块再开始写。

### GDB

GDB删除指定的断点

delete [breakpoints] [rang...]

 breakpoints为断点号 不指定断点号则表明删除所有的断点 range表示断点号的范围