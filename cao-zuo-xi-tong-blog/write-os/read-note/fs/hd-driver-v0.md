# 硬盘驱动程序



## 小结

## 2021-06-16 07:09

看硬盘驱动程序。耗时46分。

我的笔记实在太差，可以说，没有笔记。

大概20天前看明白了文件系统，现在，几乎全部忘记了。

即使写完了操作系统，仍然会忘记。

### 2021-06-16 07:09

看以前的笔记，时间主要消耗在下面这段代码：

```c
// file:/home/cg/yuyuan-os/osfs10/e/c-demo/traverse.c

#include <stdio.h>
#include <string.h>

int main(int argc, char * argv[])
{
        // world!
        char str[20] = "owlr!d";
        char res[20];
        char *p = str;
        int len = strlen(str);
        for(int i = 0; i < len / 2; i++){
                res[2*i+1] = *p++;
                res[2*i] = *p++;
        }
        res[len] = 0;

        printf("str = %s\n", str);
        printf("res = %s\n", res);

        return 0;
}
```

执行结果：

```shell
[root@localhost c-demo]# ./traverse
str = owlr!d
res = world!
```

如果str的长度不是偶数，结果是啥？我心算不出来。会丢失最后一个字符。

耗时59分。在其他笔记上也耗费了不少时间，只是花在这段代码上的时间最多而已。我到现在仍然有点纠结这个函数。