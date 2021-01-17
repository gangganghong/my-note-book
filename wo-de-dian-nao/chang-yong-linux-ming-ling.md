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

