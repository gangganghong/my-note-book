# 怎么实现系统调用

## 什么是系统调用

有些事情只有操作系统才能做，比如，获取系统内存、创建文件。用户进程想获取系统内存或创建文件，只能向操作系统求助，对操作系统说，“操作系统大神，请帮我创建一个文件”。操作系统收到用户进程的请求后会创建一个文件并将文件句柄等返回给用户进程。

在上面这个通俗例子中，用户进程通过操作系统完成特定功能，这就是系统调用。通常，系统调用的外在表现是一个函数。

再举一个更具体的例子。在`vue`中，只需要调用一个`API`就能产生一个弹窗。`API`就是`vue`的"系统调用"。

## 关键知识点

要看懂系统调用的实现过程，需理解三个知识点：

1. `eax`。无论是C语言写的函数还是汇编写的函数，返回值都会存储在`eax`这个寄存器中。
2. `typedef     void     (*ptrFunc)(int  n)`。它定义了一个新类型`ptrFunc`。这个新类型是一个函数指针。
3. `ptrFunc handlers[3]`。`handlers`是数组的第一个元素的内存地址，`handlers+4`是第二个元素的地址，`[handlers+4]`是第二个元素， `call [handlers+4]`是调用第二个元素指向的函数。每个指针占用内存空间4个字节。

## 怎么实现

### 中断向量表

完成系统调用的具体函数存储在一个函数指针类型数组中。

```c
typedef		void		(*SYS_CALL_HANDLER);

SYS_CALL_HANDLER	sys_call_handler[1] = {create};

int create()
{
  	// 创建文件，文件句柄fd是3
  	int fd = 3;		// 文件句柄
  
  	return fd;
}
```

在中断向量表中增加一个中断向量，例如`0x90`，并且设置对应的代码，在汇编中，这个代码段的形式是一个函数，例如：

```assembly
sys_call:
		; 中断，保存进程
		
		call	[sys_call_handler + eax * 4]
		; 创建文件，把文件句柄存储到eax
		; 修改堆栈中eax位置的那个数据的值是eax
		; 在恢复被中断进程的时候，文件句柄会在eax中返回。
		; 被中断进程通过eax得到系统调用的结果
		
		
		ret
		
```

`sys_call_handler`是数组`sys_call_handler`的内存位置。由于`sys_call_handler`中的每个元素都是一个函数指针，需要占用4个字节，所以，`sys_call_handler`中的每个元素的内存地址相差4个字节。

`call	[sys_call_handler + eax * 4]`中的`eax`是数组`sys_call_handler`的索引，在下文会解释`eax`的值从哪里来。

### 系统调用函数

系统调用函数是直接提供给用户进程使用的函数，让用户进程能够做只有操作系统才能做的事情。

这个函数的函数体，是一个中断，示意代码如下：

```assembly
createFile:
		mov			eax,			0
		int 		0x90
		
		ret
```

`mov eax, 0`将`eax`赋值为0，这就是`call	[sys_call_handler + eax * 4]`中的`eax`的值的来源。

### 使用系统调用

下面是使用系统调用的示意代码。

```c
int main(int argc, char **)
{
  		// 使用系统调用创建文件
  		int fd = createFile();
  
  		return 0;
}
```

现在，可以给出系统调用的全部运行流程了：

1. 调用函数`createFile`。
2. `int 0x90`产生一个中断，调用`sys_call`。
3. `eax` 是0，`call	[sys_call_handler + eax * 4]`变成`call	[sys_call_handler]`。
4. `[sys_call_handler]`是数组`sys_call_handler`的第一个元素`create`，`call	[sys_call_handler]`   变成 `call create`。
5. `call  create`会创建一个文件，并将文件句柄存储在堆栈中属于`eax`的那个位置。
6. 回到`createFile`时，堆栈中`eax`的那个位置仍然存储着文件句柄。这意味着，`createFile`的返回值是操作系统创建的文件的文件句柄。