# 实现IPC

花在IPC上的时间好像有三四天了。大部分难题都解决了，大部分细节也理解了。代码量比较大，我总觉得没有透彻理解。

写篇文章吧。

## IPC

IPC，是进程间通信。进程具有独立的地址空间和逻辑控制流，两个或多个进程协作，需要交换数据。我觉得这是IPC存在的必要性。

微内核下，系统调用也要使用IPC。

IPC的内容比较多，我尽量有条理地写出来。

## k_reenter和CPU所处的特权级

先啃硬骨头。

这一小节对我来说很硬，消耗了不少时间。

### CPU所处的特权级

CPU所处的特权级，就是CPL的值，分为0特权级、1特权级、2特权级、3特权级。为了表述方便，我把它们分别记作：ring0、ring1、ring2、ring3。

源文件中的代码，不存在特权级属性。只有GDT、LDT中的描述符所指向的内存才存在特权级属性。有人要说，选择子指向的内存也存在特权级属性啊。我的解释是，选择子直接指向描述符。这个解释似乎很不充分。

我直接给出一个结论吧：本小节所说的CPU所处的特权级，是运行某些指令时CPU所处的特权级，也就是CPL。

为什么要对这个问题啰嗦这么多？因为，我在本小节要讲的是：

1. 当k_reenter在函数sys_printx中的值等于0时，CPU在`<ring1~ring3>`特权级上执行系统调用printx。
2. 当k_reenter在函数sys_printx中的值大于0时，CPU在<ring0>特权级上执行系统调用printx。

### k_reenter

k_reenter是什么？它的数据类型是`unsigned int`，初始值是0。我用它来识别一个中断是否为重入中断。

在我的操作系统中，先执行restart，再发生中断（可能是时钟中断，也可能是系统调用中断，还可能是键盘中断）。

这是我断点调试看到的事实。模拟多进程执行流程，需要建立在这个事实上。

在restart中，会将k_reenter减去1。一个`unsigned int`类型的数，值是0，减去1之后，变成了`4,294,967,295`。

#### 用户进程使用printx

1. 已经在用户进程中，必定已经执行过restart，此时，k_reenter的值是`4,294,967,295`。
2. 使用printx，会触发系统调用中断。
3. 在系统调用中断中，会为当前进程建立快照。在建立快照的过程中，会将k_reenter加1。
4. 当进入sys_printx时，k_reenter的值是 `4,294,967,295` + 1。`unsigned int`类型的最大值加1，结果是0。
5. 所以，在sys_printx中，k_reenter的值是0时，使用printx时CPU处于用户进程中。
6. 用户进程的特权级是ring3。

将上面的步骤倒着推理一次，也成立。

1. 在sys_printx中，k_reenter的值是0，那么，在系统调用中断之前，k_reenter的值是`4,294,967,295`。
2. k_reenter的初始值是0，怎么才能让0变成`4,294,967,295`？答案是，减去1。
3. 只有在restart中，才能让k_reenter减去1。
4. 执行restart后，一定会在用户进程而不是在内核中。

如此，我得出一个结论：”k_reenter的值为0“是"调用printx时CPU处于<ring1~ring3>特权级"的充要条件。

#### 内核使用printx

更严谨的描述是，CPU使用printx时处于ring0。

1. sys_printx使用了out_char，因此，必须在tty初始化之后才能使用printx，也就是说，必须在restart执行之后才能使用。
2. 当然，你也可以在restart执行之前的内核中使用printx，可是，会报错。
3. 所以，还是回归到第1点限制的条件。
4. 执行restart后，k_reenter减去1，变成`4,294,967,295`。
5. 进入sys_printx后，k_reenter加1，因为发生了系统调用中断。
6. 注意，并非在第4步之后马上进入sys_printx。
7. 我讨论的是，从内核中进入sys_printx。
8. 目前，有哪些内核代码？时钟中断例程，即clock_handler。还有其他运行于ring0而且是在tty初始化之后的代码吗？暂时没有。
9. 既然如此，那就分析，在clock_handler中使用printx。
10. 进入clock_handler，必然发生了时钟中断。时钟中断例程中，k_reenter会加1。
11. 第5步和第10步，k_reenter共增加2。`4,294,967,295` + 2，结果是2。
12. 结论是：CPU使用printx时处于ring0，那么，k_reenter的值是2即大于0。

反过来推理呢？

1. k_reenter的初始值是`4,294,967,295`，要变成大于0的值，至少需要经过两次中断。
2. 系统调用中断一次。
3. 还有一次中断呢？可能是时钟中断，也可能是系统中断。
4. 一定是在中断例程之中调用了printx。
   1. 如果是在中断例程结束之后，k_reenter会减去1。加1又减去1，进入sys_printx时，k_reenter只会加1，最终结果是0。

模拟多进程的运行，对我来说，是一件非常非常不靠谱的事情。我觉得，上面的推理，没有可靠的依据，是我在牵强附会。

根据k_reenter判断CPU调用printx时的特权级，实在不是一个好方法。

## 调试函数

### panic

#### 魔数

在要打印的字符串的开头加上魔数A，在sys_printx中根据魔数识别是不是panic。

只要识别出是panic，就打印字符串，并且hlt。

在哪里打印？在整个显存打印字符串，因此，不能使用out_char。out_char需要指定tty。

#### 打印

在整个显存打印字符串。算法是这样的：

1. 从0xb8000开始打印，直接向显存写入数据。
2. 打印完字符串后，把接下来的16行显存的背景色设置为灰色，不改变字符。
3. 重复第2步，一直到显存耗尽。

对了，要关闭中断。因为，出现错误后，要停止整个系统。如果允许中断，就会切换到其他代码执行。

### assert

在要打印的字符串的开头加上魔数B，在sys_printx中根据魔数识别是不是assert。

和panic的差别是：识别出是assert，还需要是0特权级，才执行和panic一样的操作；否则，使用out_char打印字符串，然后正常返回，在用户进程使用spin停止用户进程。

### 相关函数

#### printl

是一个宏，对printf的封装。我不知道为何多此一举。

#### sys_printx

通过系统调用printx调用sys_printx，并不直接使用它。

在panic和assert中所写的逻辑，其实是在这里实现。

### printx

系统调用，用汇编代码编写。代码模板是：

1. 参数
   1. 实际完成系统调用功能的函数在sys_call_table中的索引。
   2. 实际完成系统调用功能的函数的参数。
2. `int 中断向量`
3. ret

## MESSAGE

这是一个结构体，用来表示消息。结构体的构成很奇特，我不理解为什么这个结构体会设计成这样。

该结构体使用了共同体，即UNION。UNION的大小是最大的成员的大小。对UNION的多个成员赋值，最后赋值的那个成员的值会覆盖前面所有的成员的赋值。

非应试，可以不必记住MESSAGE的结构。

## 消息传递

在我的操作系统中，进程有三种状态，用进程表中的flag表示：

1. 0。运行中。
2. SEDING：0x2。阻塞。因发生消息未完成而阻塞。
3. RECEIVING：0x4。阻塞。因接收消息未完成而阻塞。

消息传递，就是把消息从一个进程复制到另一个进程。对，在进程表中，需新增一个消息成员。消息的数据类型就是MESSAGE。

### send_msg

三个必备参数：

1. 发送方。
2. 接收方。
3. 消息。

> 卡了很久，才想起这些。一卡住，我就容易有杂念，就不想写了。
>
> 这似乎是在背诵。
>
> 呵呵，学习，不是背诵，莫非是创造？
>
> 管它呢，我能给出合适的、好的解决方案，管他是背诵还是创造。

重点关注接收方。我把IPC设计成同步，发送方发送消息结束前，发送方一直处于SENDING状态，是阻塞的。

接收方需要是RECEIVING状态、发送方才能向它发送消息吗？

多猜无益。我在这块的知识实在太少，没有必要自己创造。直接看笔记。

#### 业务逻辑

1. 接收方是RECEIVING状态
   1. 把发送方的消息复制到接收方。
   2. 设置发送方的状态（进程表中和IPC相关的成员）
   3. 设置接收方（进程表中和IPC相关的成员）
2. 接收方不是RECEIVING状态
   1. 把发送方放入接收方的接收队列。这个队列中存储的是进程。
   2. 设置发送方：
      1. 进程状态，设置成SENDING。
      2. 还有其他吗？
   3. 设置接收方：
      1. 设置什么？

> 这些细节，若不是真正感觉到由需求，我怎么设计出它们需要什么？
>
> 当然，能死记硬背。

### receive_msg

#### 业务逻辑

1. 是否有中断信息。
   1. 这种情况，不处理。目前，不会遇到这种情况。
2. 接收任意消息。
   1. 检查接收方的接收队列是否为空，若不为空，获取第一个元素来接收信息。
   2. 用到一个队列操作：出队头结点。很简单。
      1. 假设ptr指向接收队列，第一个元素是A。
      2. 取出A，并且设置ptr指向A的下一个元素。
3. 接收特定进程发送的消息。用到两个操作。
   1. 在接收队列中查找特定进程。
      1. 遍历，逐个比较，当找到目标进程时，记录下目标进程的前一个元素。
      2. 注意，遍历的目的，不是找目标进程，而是找到目标进程的前一个元素pre。
      3. 为何如此？在第2个操作将用到pre。
   2. 从队列中移除一个元素。
      1. 目标元素是T，前一个元素是pre，目标元素的下一个元素是B。
      2. T的下一个元素可能是0。
      3. 移除T的操作：pre->next = T->next。
4. 检查后发现能复制消息，就可以在队列中取出发送方，然后复制消息了。
5. 也可以在通过检查后把copyok赋值为true。当copyok为true时，复制消息。
6. 复制完消息后，设置发送方和接收方。
   1. 发送方：
      1. 状态：运行。
   2. 接收方：
      1. 状态：运行。
7. copyok为false，设置发送方和接收方的。
   1. 发送方：不处理，保持原状。
   2. 接收方：
      1. 状态：阻塞。

## JD

### 职位描述：

岗位描述：
1、负责一个k8s模块或一个技术方向（存储、网络、调度）的演进和推进；
2、能够主导重大产品架构演化和架构设计；
3、对技术和业务有前瞻性的思考，参与容器领域方向性的决策并推进落地。

岗位要求：
\1. 深入了解Golang/C/Java语言其一，有独立解决各种系统问题的能力，尤其对Golang语言有深刻的理解者优先考虑 ；
\2. 熟悉容器社区，对容器相关技术k8s/docker/pouch有深入了解的优化考虑；
\3. 对资源隔离有了解，对cgroup、namespace机制有深入了解，熟悉常用的资源隔离手段；
\4. 对网络、存储和进程调度有一定的了解，精通于其中至少一种的优先考虑；
\5. 对linux内核有一定了解，知道一般问题的定位方向和手段，具备Linux内核开发能力者优先。



岗位职责：
● 设计研发Shopee新一代容器化基础设施平台，承载大规模计算集群上容器应用的管理和编排，实现大规模异构算力池的高级抽象；
● 编写高质量在线系统，可维护，可审计，可复现，保障平台安全；
● 负责平台稳定性建设，将面临内核容器，大规模分布式系统领域内难得一见的技术挑战，为Shopee多个大规模集群保驾护航；
● 提高资源利用效率，降低成本，设计和研发离线在线混部，等平台技术；
● 建设智能调度，结合动态运行数据、深度学习、强化学习等技术打造下一代智能化、可视化的调度技术；
● 支持计算类、大数据类、机器学习/深度学习等业务的资源调度，设计和研发高并发、低延迟、大规模的调度技术；

岗位要求：
● 计算机科学，计算机工程，电子工程等相关专业本科以上学历；
● 具备扎实的编码功底，良好的设计能力和开发习惯，有一定的生产运维经验，熟悉DevOps研发流程；
● 精通Go/Java/C++，至少熟悉一门脚本语言，5年以上一线研发经验；
● 对Linux操作系统有全面认识，熟练Linux操作；
● 3年以上相关云原生和分布式系统工作经验，熟悉k8s架构，灵活扩展k8s系统；
● 具有黑客精神，打破成规，突破边界；
● 较强的团队沟通和协作能力，较强的自我驱动能力。

https://www.lagou.com/jobs/8548174.html


优先相关经验：
● 掌握K8S，docker等云原生领域知识，对云原生开源社区有贡献；
● 有云计算（阿里云，腾讯云，AWS，Azure，GCP，华为云等）或大流量互联网平台基础设施或分布式系统设计开发经验；
● 大规模项目经验，拥有全栈视野。

https://www.lagou.com/jobs/8384161.html



**职位诱惑：**

平台大，发展快，待遇好，福利棒

### 职位描述：

岗位名称：腾讯云 容器开发高级工程师
职级参考：T9-12
工作地点：北京、上海、深圳、成都
所属部门：腾讯云
招聘数量：5

岗位要求：
\1. 8 年以上后端开发经验，Coding、Debug 能力强， 有丰富的架构设计经验。熟悉Linux，包括Linux网络和常用的服务等；
\2. 熟悉 C/C++/Go/Java/Python/Ruby 等至少二种编程语言； 
\3. 熟悉 Docker/Kubernetes/Swarm/Mesos 等技术；
\4. 熟悉 Jenkins/ELK/Prometheus 等技术优先；
\5. 熟悉 云计算厂商产品优先。 

岗位职责：
\1. 负责腾讯云公有云/私有云中 Kubernetes/Devops/Servicemesh 等产品技术方案设计与研发工作；
\2. 负责 Kubernetes 相关前沿技术规划和调研工作，从技术上保证产品的竞争力；
\3. 负责与产品及客户沟通，判定需求的合理性，找出最合适的方式解决客户问题； 

https://www.lagou.com/jobs/6984986.html?show=5c06f12dfbf44d069763c422740c100f

http://www.wenxue100.com/book_EnglishBook/bookList.aspx

chuganghong

2021cslh

## 总结

### 2021-05-01 22:04

消耗1个小时33分。写k_reenter的值和调用printx时CPU所处的特权级。价值很低。这并不是应用广泛的知识。推理，不严谨，不可靠。

我将继续寻找更好的检测中断重入的方法，也将寻找更好的实现assert的方法。在我写完这一期操作系统的所有功能之后。

不可再在这类知识点耗费这么多时间。什么知识点？应用范围小，不是好方法。

### 2021-05-01 11:28

消耗22分。我却感觉过了很久。

复述send_msg和receive_msg，并非百分百背诵，我记住的知识点没那么多。还原度非常低，信息损耗非常大。

我该怎么办？

1. 按我记住的这些写代码。
2. 对照着书上的写代码。
3. 重复看书，直到我能百分百复述书中的内容。这就是背诵。

按第三种做法。我一直都是这么做的。

第二种做法，是在抄写。

第一种，所学知识太少，低水平。