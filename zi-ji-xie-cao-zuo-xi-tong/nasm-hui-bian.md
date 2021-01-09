# NASM汇编

最新nams汇编官方文档，有64位代码，太棒了！

\`DB'一类的伪指令: 声明已初始化的数据。

```assembly
db    0x55                ; just the byte 0x55
db    0x55,0x56,0x57      ; three bytes in succession
db    'a',0x55            ; character constants are OK
db    'hello',13,10,'$'   ; so are string constants
dw    0x1234              ; 0x34 0x12
dw    'a'                 ; 0x41 0x00 (it's just a number)
dw    'ab'                ; 0x41 0x42 (character constant)
dw    'abc'               ; 0x41 0x42 0x43 0x00 (string)
dd    0x12345678          ; 0x78 0x56 0x34 0x12
dd    1.234567e20         ; floating-point constant
dq    1.234567e20         ; double-precision float
dt    1.234567e20         ; extended-precision float
```

'DQ'和'DT'不接受数值常数或字符串常数作为操作数。

\`RESB'类的伪指令: 声明未初始化的数据。

`RESB',`RESW', `RESD',`RESQ' and \`REST'被设计用在模块的BSS段中：它们声明 未初始化的存储空间。每一个带有单个操作数，用来表明字节数，字数，或双字数 或其他的需要保留单位。

```text
buffer:         resb    64              ; reserve 64 bytes
wordvar:        resw    1               ; reserve a word
realarray       resq    10              ; array of ten reals
```

\`INCBIN':包含其他二进制文件

这能很方便地在一个游戏可执行文件中包含中图像或声音数 据。它可以以下三种形式的任何一种使用：

```text
      incbin  "file.dat"             ; include the whole file
      incbin  "file.dat",1024        ; skip the first 1024 bytes
      incbin  "file.dat",1024,512    ; skip the first 1024, and
                                     ; actually include at most 512
```

\`EQU': 定义常数。

'EQU'定义一个符号，代表一个常量值：当使用'EQU'时，源文件行上必须包含一个label。 'EQU'的行为就是把给出的label的名字定义成它的操作数\(唯一\)的值。定义是不可更 改的，比如：

```text
message db 'hello, world'
msglen equ $-message
```

把'msglen'定义成了常量12。'msglen'不能再被重定义。这也不是一个预自理定义： 'msglen'的值只被计算一次，计算中使用到了'$'\(参阅3.5\)在此时的含义。

\`TIMES': 重复指令或数据。

为了与绝大多数C编译器的Makefile保持兼容，该选项也可以被写成'-U'。

\`-e'选项: 仅预处理。

NASM允许预处理器独立运行。使用'-e'选项\(不需要参数\)会导致NASM预处理输入 文件，展开所有的宏，去掉所有的注释和预处理操作符，然后把结果文件打印在标 准输出上\(如果'-o'选项也被指定的话，会被存入一个文件\)。

该选项不能被用在那些需要预处理器去计算与符号相关的表达式的程序中，所以 如下面的代码：

```text
  %assign tablesize ($-tablestart)
```

会在仅预处理模式中会出错。（不理解）

\`-On'选项: 指定多遍优化。（不理解）

NASM在缺省状态下是一个两遍的汇编器。这意味着如果你有一个复杂的源文件需要 多于两遍的汇编。你必须告诉它。

\`-t'选项: 使用TASM兼容模式。

\`-w'选项: 使汇编警告信息有效或无效。

输入'NASM -v'会显示你正使用的NASM的版本号，还有它被编译的时间。

\`NASMENV'环境变量。\(不理解）

NASM是大小写敏感的。

NASM需要方括号来引用内存地址。

NASM不存储变量的类型。

必须显式地写上代码'mov word \[var\],2'。

因为这个原因，NASM不支持'LODS','MOVS','STOS','SCANS','CMPS','INS',或'OUTS' 指令，仅仅支持形如'LODSB','MOVSW',和'SCANSD'之类的指令。它们都显式地指定 被处理的字符串的尺寸。

NASM不会 \`ASSUME'

NASM不支持内存模型。

NASM同样不含有任何操作符来支持不同的16位内存模型。程序员需要自己跟踪那 些函数需要far call，哪些需要near call。并需要确定放置正确的'RET'指令\('RETN' 或'RETF'; NASM接受'RET'作为'RETN'的另一种形式\);另外程序员需要在调用外部函 数时在需要的编写CALL FAR指令，并必须跟踪哪些外部变量定义是far,哪些是near。

浮点处理上的不同。

NASM使用跟MASM不同的浮点寄存器名：MASM叫它们'ST\(0\)','ST\(1\)'等，而'a86'叫它们 '0','1'等，NASM则叫它们'st0','st1'等。（不理解）

`label: instruction operands ; comment`

label,instruction,comment存在或不存在都是允 许的。当然，operands域会因为instruction域的要求而必需存或必须不存在。

NASM使用反斜线\(\)作为续行符；如果一个以一个反斜线结束，那第二行会被认为 是前面一行的一部分。

NASM对于一行中的空格符并没有严格的限制：labels可以在它们的前面有空格，或 其他任何东西。label后面的冒号同样也是可选的。\(注意到，这意味着如果你想 要写一行'lodsb'，但却错误地写成了'lodab'，这仍将是有效的一行，但这一行不做 任何事情，只是定义了一个label。运行NASM时带上命令行选项'-w+orphan-labels' 会让NASM在你定义了一个不以冒号结尾的label时警告你。

labels中的有效的字符是字母，数字，'-','$','\#','@','~','.'和'?'。但只有字母 '.',\(具有特殊含义，参阅3.9\),'\_'和'?'可以作为标识符的开头。一个标识符还可 以加上一个'$'前缀，以表明它被作为一个标识符而不是保留字来处理。这样的话， 如果你想到链接进来的其他模块中定义了一个符号叫'eax'，你可以用'$eax'在 NASM代码中引用它，以和寄存器的符号区分开。

nstruction域可以包含任何机器指令：Pentium和P6指令，FPU指令，MMX指令还有甚 至没有公开的指令也会被支持。这些指令可以加上前缀'LOCK','REP','REPE/REPZ' 或'REPNE'/'REPNZ'，通常，支持显示的地址尺寸和操作数尺寸前缀'A16','A32', 'O16'和'O32'。关于使用它们的一个例子在第九章给出。你也可以使用段寄存器 名作为指令前缀： 代码'es mov \[bx\],ax'等效于代码'mov \[es:bx\],ax'。我们推荐 后一种语法。因为它和语法中的其它语法特性一致。但是对于象'LODSB'这样的 指令，它没有操作数，但还是可以有一个段前缀， 对于'es lodsb'没有清晰地语法 处理方式

在使用一个前缀时，指令不是必须的，像'CS','A32','LOCK'或'REPE'这样的段前缀 可以单独出现在一行上，NASM仅仅产生一个前缀字节。\(不理解）

指令操作数可以使用一定的格式：它们可以是寄存器，仅仅以寄存器名来表示\(比 如：'ax','bp','ebx','cr0'：NASM不使用'gas'的语法风格，在这种风格中，寄存器名 前必须加上一个'%'符号\)，或者它们可以是有效的地址\(参阅3.3\)，常数\(3.4\)，或 表达式。

对于浮点指令，NASM接受各种语法：你可以使用MASM支持的双操作数形式，或者你 可以使用NASM的在大多数情况下全用的单操作数形式。支持的所以指令的语法 细节可以参阅附录B。比如，你可以写：

```text
          fadd    st1             ; this sets st0 := st0 + st1
          fadd    st0,st1         ; so does this

          fadd    st1,st0         ; this sets st1 := st1 + st0
          fadd    to st1          ; so does this
```

几乎所有的浮点指令在引用内存时必须使用以下前缀中的一个'DWORD',QWORD' 或'TWORD'来指明它所引用的内存的尺寸。（不理解）

\`TIMES': 重复指令或数据。

前缀'TIMES'导致指令被汇编多次。它在某种程序上是NASM的与MASM兼容汇编器的 'DUP'语法的等价物。你可以这样写：

zerobuf: times 64 db 0

或类似的东西，但'TEIMES'的能力远不止于此。'TIMES'的参数不仅仅是一个数值常 数，还有数值表达式，所以你可以这样做：

buffer: db 'hello, world' times 64-$+buffer db ' '

它可以把'buffer'的长度精确地定义为64字节，’TIMES‘可以被用在一般地指令上， 所以你可像这要编写不展开的循环：

```text
          times 100 movsb
```

注意在'times 100 resb 1'跟'resb 100'之间并没有显著的区别，除了后者在汇编 时会快上一百倍。

就像'EQU','RESB'它们一样， 'TIMES'的操作数也是严格语法的表达式。\(见3.8\)

注意'TIMES'不可以被用在宏上：原因是'TIMES'在宏被分析后再被处理，它允许 ’TIMES'的参数包含像上面的'64-$+buffer'这样的表达式。要重复多于一行的代 码，或者一个宏，使用预处理指令'%rep'。

有效地址

一个有效地址是一个指令的操作数,它是对内存的一个引用。在NASM中，有效地址 的语法是非常简单的：它由一个可计算的表达式组成，放在一个中括号内。比如：

```text
  wordvar dw      123
          mov     ax,[wordvar]
          mov     ax,[wordvar+1]
          mov     ax,[es:wordvar+bx]
```

任何与上例不一致的表达都不是NASM中有效的内存引用，比如：'es:wordvar\[bx\]'。

更复杂一些的有效地址，比如含有多个寄存器的，也是以同样的方式工作：

```text
          mov     eax,[ebx*2+ecx+offset]
          mov     ax,[bp+di+8]
```

NASM在这些有效地址上具有进行代数运算的能力，所以看似不合法的一些有效地址 使用上都是没有问题的：

```text
      mov     eax,[ebx*5]             ; assembles as [ebx*4+ebx]
      mov     eax,[label1*2-label2]   ; ie [label1+(label1-label2)]
```

有些形式的有效地址在汇编后具有多种形式；在大多数情况下，NASM会自动产生 最小化的形式。比如，32位的有效地址'\[eax\*2+0\]'和'\[eax+eax\]'在汇编后具有 完全不同的形式，NASM通常只会生成后者，因为前者会为0偏移多开辟4个字节。

NASM具有一种隐含的机制，它会对'\[eax+ebx\]'和'\[ebx+eax\]'产生不同的操作码； 通常，这是很有用的，因为'\[esi+ebp\]'和'\[ebp+esi\]'具有不同的缺省段寄存器。

尽管如此，你也可以使用关键字'BYTE','WORD','DWORD'和'NOSPLIT'强制NASM产 生特定形式的有效地址。如果你想让'\[eax+3\]'被汇编成具有一个double-word的 偏移域，而不是由NASM缺省产生一个字节的偏移。你可以使用'\[dword eax+3\]'， 同样，你可以强制NASM为一个第一遍汇编时没有看见的小值产生一个一字节的偏 移\(像这样的例子，可以参阅3.8\)。比如：'\[byte eax+offset\]'。有一种特殊情 况，‘\[byte eax\]'会被汇编成'\[eax+0\]'。带有一个字节的0偏移。而'\[dword eax\]'会带一个double-word的0偏移。而常用的形式，'\[eax\]'则不会带有偏移域。

当你希望在16位的代码中存取32位段中的数据时，上面所描述的形式是非常有用 的。关于这方面的更多信息，请参阅9.2。实际上，如果你要存取一个在已知偏 移地址处的数据，而这个地址又大于16位值，如果你不指定一个dword偏移， NASM会让高位上的偏移值丢失。

类似的，NASM会把'\[eax_2\]'分裂成'\[eax+eax\]' ,因为这样可以让偏移域不存在 以此节省空间；实际上，它也把'\[eax_2+offset\]'分成'\[eax+eax+offset\]'，你 可以使用‘NOSPLIT'关键字改变这种行为：`[nosplit eax*2]'会强制`\[eax\*2+0\]'按字面意思被处理。（不理解）

常数

NASM能理解四种不同类型的常数：数值，字符，字符串和浮点数。

数值常数。

一个数值常数就只是一个数值而已。NASM允许你以多种方式指定数值使用的 进制，你可以以后缀'H','Q','B'来指定十六进制数，八进制数和二进制数， 或者你可以用C风格的前缀'0x'表示十六进制数，或者以Borland Pascal风 格的前缀'$'来表示十六进制数，注意，'$'前缀在标识符中具有双重职责 \(参阅3.1\)，所以一个以'$'作前缀的十六进制数值必须在'$'后紧跟数字，而 不是字符。

请看一些例子：

```text
          mov     ax,100          ; decimal
          mov     ax,0a2h         ; hex
          mov     ax,$0a2         ; hex again: the 0 is required
          mov     ax,0xa2         ; hex yet again
          mov     ax,777q         ; octal
          mov     ax,10010011b    ; binary
```

字符型常数。

一个字符常数最多由包含在双引号或单引号中的四个字符组成。引号的类型 与使用跟NASM其它地方没什么区别，但有一点，单引号中允许有双引号出现。

一个具有多个字符的字符常数会被little-endian order，如果你编写：

```text
            mov eax,'abcd'

  产生的常数不会是`0x61626364'，而是`0x64636261'，所以你把常数存入内存
  的话，它会读成'abcd'而不是'dcba'。
```

字符串常数。

字符串常数一般只被一些伪操作指令接受，比如'DB'类，还有'INCBIN'。

一个字符串常数和字符常数看上去很相像，但会长一些。它被处理成最大长 度的字符常数之间的连接。所以，以下两个语句是等价的：

```text
        db    'hello'               ; string constant
        db    'h','e','l','l','o'   ; equivalent character constants
```

还有，下面的也是等价的:

```text
        dd    'ninechars'           ; doubleword string constant
        dd    'nine','char','s'     ; becomes three doublewords
        db    'ninechars',0,0,0     ; and really looks like this
```

注意，如果作为'db'的操作数，类似'ab'的常数会被处理成字符串常量，因 为它作为字符常数的话，还不够短，因为，如果不这样，那'db 'ab'会跟 'db 'a''具有同样的效果，那是很愚蠢的。同样的，三字符或四字符常数会 在作为'dw'的操作数时被处理成字符串。（不理解）

浮点常量

浮点常量只在作为'DD','DQ','DT'的操作数时被接受。它们以传统的形式表 达：数值，然后一个句点，然后是可选的更多的数值，然后是选项'E'跟上 一个指数。句点是强制必须有的，这样，NASM就可以把它们跟'dd 1'区分开， 它只是声明一个整型常数，而'dd 1.0'声明一个浮点型常数。

一些例子:

```text
        dd    1.2                     ; an easy one
        dq    1.e10                   ; 10,000,000,000
        dq    1.e+10                  ; synonymous with 1.e10
        dq    1.e-10                  ; 0.000 000 000 1
        dt    3.141592653589793238462 ; pi
```

NASM不能在编译时求浮点常数的值。这是因为NASM被设计为可移植的，尽管它 常产生x86处理器上的代码，汇编器本身却可以和ANSI C编译器一起运行在任 何系统上。所以，汇编器不能保证系统上总存在一个能处理Intel浮点数的浮 点单元。所以，NASM为了能够处理浮点运算，它必须含有它自己的一套完整 的浮点处理例程，它大大增加了汇编器的大小，却获得了并不多的好处。

表达式

NASM中的表达式语法跟C里的是非常相似的。

NASM不能确定编译时在计算表达式时的整型数尺寸：因为NASM可以在64位系 统上非常好的编译和运行，不要假设表达式总是在32位的寄存器中被计算的， 所以要慎重地对待整型数溢出的情况。它并不总能正常的工作。NASM唯一能 够保证的是：你至少拥有32位长度。

NASM在表达式中支持两个特殊的记号，即'$'和'$$',它们允许引用当前指令 的地址。'$'计算得到它本身所在源代码行的开始处的地址；所以你可以简 单地写这样的代码'jmp $'来表示无限循环。'$$'计算当前段开始处的地址， 所以你可以通过\($-$$\)找出你当前在段内的偏移。

NASM提供的运算符以运算优先级为序列举如下：

3.5.1 \`\|': 位或运算符。

运算符'\|'给出一个位级的或运算，所执行的操作与机器指令'or'是完全相 同的。位或是NASM中优先级最低的运算符。

3.5.2 \`^': 位异或运算符。

```text
  `^' 提供位异或操作。
```

3.5.3 \`&': 位与运算符。

```text
  `&' 提供位与运算。
```

3.5.4 `<<' and`&gt;&gt;': 位移运算符。

```text
  `<<' 提供位左移, 跟C中的实现一样，所以'5<<3'相当于把5乘上8。'>>'提
  供位右移。在NASM中，这样的位移总是无符号的，所以位移后，左侧总是以
  零填充，并不会有符号扩展。
```

3.5.5 `+' and`-': 加与减运算符。

'+'与'-'运算符提供完整的普通加减法功能。

3.5.6 `*',`/', `//',`%'和\`%%': 乘除法运算符。

'\*'是乘法运算符。'/'和'//'都是除法运算符，'/'是无符号除，'//'是带 符号除。同样的，'%'和'%%'提供无符号与带符号的模运算。

同ANSI C一样，NASM不保证对带符号模操作执行的操作的有效性。

因为'%'符号也被宏预处理器使用，你必须保证不管是带符号还是无符号的 模操作符都必须跟有空格。

3.5.7 一元运算符: `+',`-', `~'和`SEG'

这些只作用于一个参数的一元运算符是NASM的表达式语法中优先级最高的。 '-'把它的操作数取反，'+'不作任何事情\(它只是为了和'-'保持对称\), '~'对它的操作数取补码，而'SEG'提供它的操作数的段地址\(在3.6中会有 详细解释\)。

3.6 `SEG'和`WRT'

当写很大的16位程序时，必须把它分成很多段，这时，引用段内一个符号的 地址的能力是非常有必要的，NASM提供了'SEG'操作符来实现这个功能。

'SEG'操作符返回符号所在的首选段的段基址，即一个段基址，当符号的偏 移地址以它为参考时，是有效的，所以，代码：

```text
          mov     ax,seg symbol
          mov     es,ax
          mov     bx,symbol
```

总是在'ES:BX'中载入一个指向符号'symbol'的有效指针。

而事情往往可能比这还要复杂些：因为16位的段与组是可以相互重叠的， 你通常可能需要通过不同的段基址，而不是首选的段基址来引用一个符 号，NASM可以让你这样做，通过使用'WRT'关键字，你可以这样写：

```text
          mov     ax,weird_seg        ; weird_seg is a segment base
          mov     es,ax
          mov     bx,symbol wrt weird_seg
```

会在'ES:BX'中载入一个不同的，但功能上却是相同的指向'symbol'的指 针。

通过使用'call segment:offset'，NASM提供fall call\(段内\)和jump,这里 'segment'和'offset'都以立即数的形式出现。所以要调用一个远过程，你 可以如下编写代码：

```text
          call    (seg procedure):procedure
          call    weird_seg:(procedure wrt weird_seg)
```

\(上面的圆括号只是为了说明方便，实际使用中并不需要\)

NASM支持形如'call far procedure'的语法，跟上面第一句是等价的。'jmp' 的工作方式跟'call'在这里完全相同。

在数据段中要声明一个指向数据元素的远指针，可以象下面这样写：

```text
          dw      symbol, seg symbol
```

NASM没有提供更便利的写法，但你可以用宏自己建造一个。

`SEG'和`WRT'

当写很大的16位程序时，必须把它分成很多段，这时，引用段内一个符号的 地址的能力是非常有必要的，NASM提供了'SEG'操作符来实现这个功能。

'SEG'操作符返回符号所在的首选段的段基址，即一个段基址，当符号的偏 移地址以它为参考时，是有效的，所以，代码：

```text
          mov     ax,seg symbol
          mov     es,ax
          mov     bx,symbol
```

总是在'ES:BX'中载入一个指向符号'symbol'的有效指针。

而事情往往可能比这还要复杂些：因为16位的段与组是可以相互重叠的， 你通常可能需要通过不同的段基址，而不是首选的段基址来引用一个符 号，NASM可以让你这样做，通过使用'WRT'关键字，你可以这样写：

```text
          mov     ax,weird_seg        ; weird_seg is a segment base
          mov     es,ax
          mov     bx,symbol wrt weird_seg
```

会在'ES:BX'中载入一个不同的，但功能上却是相同的指向'symbol'的指 针。

通过使用'call segment:offset'，NASM提供fall call\(段内\)和jump,这里 'segment'和'offset'都以立即数的形式出现。所以要调用一个远过程，你 可以如下编写代码：

```text
          call    (seg procedure):procedure
          call    weird_seg:(procedure wrt weird_seg)
```

\(上面的圆括号只是为了说明方便，实际使用中并不需要\)

NASM支持形如'call far procedure'的语法，跟上面第一句是等价的。'jmp' 的工作方式跟'call'在这里完全相同。

在数据段中要声明一个指向数据元素的远指针，可以象下面这样写：

```text
          dw      symbol, seg symbol
```

NASM没有提供更便利的写法，但你可以用宏自己建造一个。（不理解）

\`STRICT': 约束优化。

当在汇编时把优化器打开到2或更高级的时候\(参阅2.1.15\)。NASM会使用 尺寸约束\('BYTE','WORD','DWORD','QWORD',或'TWORD'\)，会给它们尽可 能小的尺寸。关键字'STRICT'用来制约这种优化，强制一个特定的操作 数为一个特定的尺寸。比如，当优化器打开，并在'BITS 16'模式下：

```text
          push dword 33

  会被编码成 `66 6A 21',而

          push strict dword 33
```

会被编码成六个字节，带有一个完整的双字立即数\`66 68 21 00 00 00'.

而当优化器关闭时，不管'STRICT'有没有使用，都会产生相同的代码。（不理解）

3.8 临界表达式。

NASM的一个限制是它是一个两遍的汇编器；不像TASM和其它汇编器，它总是 只做两遍汇编。所以它就不能处理那些非常复杂的需要三遍甚至更多遍汇编 的源代码。

第一遍汇编是用于确定所有的代码与数据的尺寸大小，这样的话，在第二遍 产生代码的时候，就可以知道代码引用的所有符号地址。所以，有一件事 NASM不能处理，那就是一段代码的尺寸依赖于另一个符号值，而这个符号又 在这段代码的后面被声明。比如：

```text
          times (label-$) db 0
        label:  db      'Where am I?'
```

'TIMES'的参数本来是可以合法得进行计算的，但NASM中不允许这样做，因为 它在第一次看到TIMES时的时候并不知道它的尺寸大小。它会拒绝这样的代码。

```text
          times (label-$+1) db 0
          label:  db      'NOW where am I?'
```

在上面的代码中，TIMES的参数是错误的。

NASM使用一个叫做临界表达式的概念，以禁止上述的这些例子，临界表达式 被定义为一个表达式，它所需要的值在第一遍汇编时都是可计算的，所以， 该表达式所依赖的符号都是之前已经定义了的，'TIMES'前缀的参数就是一个 临界表达式；同样的原因，'RESB'类的伪指令的参数也是临界表达式。

临界表达式可能会出现下面这样的情况：

```text
                   mov     ax,symbol1
   symbol1         equ     symbol2
   symbol2:
```

在第一遍的时候，NASM不能确定'symbol1'的值，因为'symbol1'被定义成等于 'symbols2',而这时，NASM还没有看到symbol2。所以在第二遍的时候，当它遇 上'mov ax,symbol1',它不能为它产生正确的代码，因为它还没有知道'symbol1' 的值。当到达下一行的时候，它又看到了'EQU'，这时它可以确定symbol1的值 了，但这时已经太晚了。

NASM为了避免此类问题，把'EQU'右侧的表达式也定义为临界表达式，所以， 'symbol1'的定义在第一遍的时候就会被拒绝。

这里还有一个关于前向引用的问题：考虑下面的代码段：

```text
          mov     eax,[ebx+offset]
  offset  equ     10
```

NASM在第一遍的时候，必须在不知道'offset'值的情况下计算指令 'mov eax,\[ebx+offset\]'的尺寸大小。它没有办法知道'offset'足够小，足以 放在一个字节的偏移域中，所以，它以产生一个短形式的有效地址编码的方 式来解决这个问题；在第一遍中，它所知道的所有关于'offset'的情况是：它 可能是代码段中的一个符号，而且，它可能需要四字节的形式。所以，它强制 这条指令的长度为适合四字节地址域的长度。在第二遍的时候，这个决定已经 作出了，它保持使这条指令很长，所以，这种情况下产生的代码没有足够的小， 这个问题可以通过先定义offset的办法得到解决，或者强制有效地址的尺寸大 小，象这样写代码： \[byte ebx+offset\]。（不理解）

本地Labels

NASM对于那些以一个句点开始的符号会作特殊处理,一个以单个句点开始的 Label会被处理成本地label, 这意味着它会跟前面一个非本地label相关联. 比如:

```text
  label1  ; some code

  .loop
          ; some more code

          jne     .loop
          ret

  label2  ; some code

  .loop
          ; some more code

          jne     .loop
          ret
```

上面的代码片断中,每一个'JNE'指令跳至离它较近的前面的一行上,因为'.loop' 的两个定义通过与它们前面的非本地Label相关联而被分离开来了。

对于本地Label的处理方式是从老的Amiga汇编器DevPac中借鉴过来的；尽管 如此，NASM提供了进一步的性能，允许从另一段代码中调用本地labels。这 是通过在本地label的前面加上非本地label前缀实现的：第一个.loop实际上被 定义为'label1.loop'，而第二个符号被记作'label2.loop'。所以你确实需要 的话你可写：

```text
  label3  ; some more code
           ; and some more

           jmp label1.loop
```

有时，这是很有用的\(比如在使用宏的时候\)，可以定义一个label,它可以 在任何地方被引用，但它不会对常规的本地label机制产生干扰。这样的 label不能是非本地label,因为非本地label会对本地labels的重复定义与 引用产生干扰；也不能是本地的，因为这样定义的宏就不能知道label的全 称了。所以NASM引进了第三类label,它只在宏定义中有用：如果一个label 以一个前缀'..@'开始，它不会对本地label产生干扰，所以，你可以写：

```text
  label1:                         ; a non-local label
  .local:                         ; this is really label1.local
  ..@foo:                         ; this is a special symbol
  label2:                         ; another non-local label
  .local:                         ; this is really label2.local

          jmp     ..@foo          ; this will jump three lines up
```

NASM还能定义其他的特殊符号，比如以两个句点开始的符号，比如 '..start'被用来指定'.obj'输出文件的执行入口。\(参阅6.2.6\)

NASM预处理器

4.1 单行的宏。

4.1.1 最常用的方式: \`%define'

单行的宏是以预处理指令'%define'定义的。定义工作同C很相似，所以你可 以这样做：

```text
  %define ctrl    0x1F &
  %define param(a,b) ((a)+(a)*(b))

          mov     byte [param(2,ebx)], ctrl 'D'
```

会被扩展为：

```text
          mov     byte [(2)+(2)*(ebx)], 0x1F & 'D'
```

当单行的宏被扩展开后还含有其它的宏时，展开工作会在执行时进行，而不是 定义时，如下面的代码：

%define a\(x\) 1+b\(x\) %define b\(x\) 2\*x

```text
          mov     ax,a(8)
```

会如预期的那样被展开成'mov ax, 1+2\*8'， 尽管宏'b'并不是在定义宏a 的时候定义的。

用'%define'定义的宏是大小写敏感的：在代码'%define foo bar'之后，只有 'foo'会被扩展成'bar'：'Foo'或者'FOO'都不会。用'%idefine'来代替'%define' \(i代表'insensitive'\)，你可以一次定义所有的大小写不同的宏。所以 '%idefine foo bar'会导致'foo','FOO','Foo'等都会被扩展成'bar'。

当一个嵌套定义\(一个宏定义中含有它本身\)的宏被展开时，有一个机制可以 检测到，并保证不会进入一个无限循环。如果有嵌套定义的宏，预处理器只 会展开第一层，因此，如果你这样写：

%define a\(x\) 1+a\(x\)

```text
          mov     ax,a(3)

  宏 `a(3)'会被扩展成'1+a(3)'，不会再被进一步扩展。这种行为是很有用的，有
```

关这样的例子请参阅8.1。

你甚至可以重载单行宏：如果你这样写：

%define foo\(x\) 1+x %define foo\(x,y\) 1+x\*y

预处理器能够处理这两种宏调用，它是通过你传递的参数的个数来进行区分的， 所以'foo\(3\)'会变成'1+3'，而'foo\(ebx,2\)'会变成'1+ebx\*2'。尽管如此，但如果 你定义了：

```text
  %define foo bar
```

那么其他的对'foo'的定义都不会被接受了：一个不带参数的宏定义不允许 对它进行带有参数进行重定义。

但这并不能阻止单行宏被重定义：你可以像这样定义，并且工作得很好：

```text
  %define foo bar
```

然后在源代码文件的稍后位置重定义它：

%define foo baz

然后，在引用宏'foo'的所有地方，它都会被扩展成最新定义的值。这在用 '%assign'定义宏时非常有用\(参阅4.1.5\)

你可以在命令行中使用'-d'选项来预定义宏。参阅2.1.11

4.1.2 %define的增强版: \`%xdefine'

与在调用宏时展开宏不同，如果想要调用一个嵌入有其他宏的宏时，使用 它在被定义的值，你需要'%define'不能提供的另外一种机制。解决的方案 是使用'%xdefine'，或者它的大小写不敏感的形式'%xidefine'。

假设你有下列的代码：

%define isTrue 1 %define isFalse isTrue %define isTrue 0

```text
  val1:    db      isFalse

  %define  isTrue  1

  val2:    db      isFalse
```

在这种情况下，'val1'等于0,而'val2'等于1。这是因为，当一个单行宏用 '%define'定义时，它只在被调用时进行展开。而'isFalse'是被展开成 'isTrue'，所以展开的是当前的'isTrue'的值。第一次宏被调用时，'isTrue' 是0,而第二次是1。

如果你希望'isFalse'被展开成在'isFalse'被定义时嵌入的'isTrue'的值， 你必须改写上面的代码，使用'%xdefine':

```text
  %xdefine isTrue  1
  %xdefine isFalse isTrue
  %xdefine isTrue  0

  val1:    db      isFalse

  %xdefine isTrue  1

  val2:    db      isFalse
```

现在每次'isFalse'被调用，它都会被展开成1，而这正是嵌入的宏'isTrue' 在'isFalse'被定义时的值。

4.1.3 : 连接单行宏的符号： \`%+'

一个单行宏中的单独的记号可以被连接起来，组成一个更长的记号以 待稍后处理。这在很多处理相似的事情的相似的宏中非常有用。

```text
  举个例子，考虑下面的代码:

  %define BDASTART 400h                ; Start of BIOS data area

  struc   tBIOSDA                      ; its structure
          .COM1addr       RESW    1
          .COM2addr       RESW    1
          ; ..and so on
  endstruc

  现在，我们需要存取tBIOSDA中的元素，我们可以这样：

          mov     ax,BDASTART + tBIOSDA.COM1addr
          mov     bx,BDASTART + tBIOSDA.COM2addr
```

如果在很多地方都要用到，这会变得非常的繁琐无趣，但使用下面 的宏会大大减小打字的量：

```text
  ; Macro to access BIOS variables by their names (from tBDA):

  %define BDA(x)  BDASTART + tBIOSDA. %+ x

  现在，我们可以象下面这样写代码：

          mov     ax,BDA(COM1addr)
          mov     bx,BDA(COM2addr)
```

使用这个特性，我们可以简单地引用大量的宏。\(另外，还可以减少打 字错误\)。

4.1.4 取消宏定义: \`%undef'

单行的宏可以使用'%undef'命令来取消。比如，下面的代码：

```text
  %define foo bar
  %undef  foo

          mov     eax, foo
```

会被展开成指令'mov eax, foo'，因为在'%undef'之后，宏'foo'处于无定义 状态。

那些被预定义的宏可以通过在命令行上使用'-u'选项来取消定义，参阅 2.1.12。

4.1.5 预处理器变量 : \`%assign'

定义单行宏的另一个方式是使用命令'%assign'\(它的大小写不敏感形式 是%iassign，它们之间的区别与'%idefine','%idefine'之间的区别完全相 同\)。

'%assign'被用来定义单行宏，它不带有参数，并有一个数值型的值。它的 值可以以表达式的形式指定，并要在'%assing'指令被处理时可以被一次 计算出来，

就像'%define','%assign'定义的宏可以在后来被重定义，所以你可以这 样做：

```text
  %assign i i+1
```

以此来增加宏的数值

'%assing'在控制'%rep'的预处理器循环的结束条件时非常有用：请参 阅4.5的例子。另外的关于'%assign'的使用在7.4和8.1中的提到。

赋给'%assign'的表达式也是临界表达式\(参阅3.8\)，而且必须可被计算 成一个纯数值型\(不能是一个可重定位的指向代码或数据的地址，或是包 含在寄存器中的一个值。\)

4.2.1 求字符串长度: \`%strlen'

'%strlen'宏就像'%assign'，会为宏创建一个数值型的值。不同点在于 '%strlen'创建的数值是一个字符串的长度。下面是一个使用的例子：

```text
  %strlen charcnt 'my string'
```

在这个例子中，'charcnt'会接受一个值8,就跟使用了'%assign'一样的 效果。在这个例子中，'my string'是一个字面上的字符串，但它也可以 是一个可以被扩展成字符串的单行宏，就像下面的例子：

```text
  %define sometext 'my string'
  %strlen charcnt sometext
```

就像第一种情况那样，这也会给'charcnt'赋值8。（为何是8？）

4.2.2 取子字符串: \`%substr'

字符串中的单个字符可以通过使用'%substr'提取出来。关于它使用的 一个例子可能比下面的描述更为有用：

```text
  %substr mychar  'xyz' 1         ; equivalent to %define mychar 'x'
  %substr mychar  'xyz' 2         ; equivalent to %define mychar 'y'
  %substr mychar  'xyz' 3         ; equivalent to %define mychar 'z'
```

在这个例子中，mychar得到了值'z'。就像在'%strlen'\(参阅4.2.1\)中那样， 第一个参数是一个将要被创建的单行宏，第二个是字符串，第三个参数 指定哪一个字符将被选出。注意，第一个索引值是1而不是0，而最后一 个索引值等同于'%strlen'给出的值。如果索引值超出了范围，会得到 一个空字符串。

4.3 多行宏: \`%macro'

多行宏看上去更象MASM和TASM中的宏：一个NASM中定义的多行宏看上去就 象下面这样：

%macro prologue 1

```text
          push    ebp
          mov     ebp,esp
          sub     esp,%1

  %endmacro

  这里，定义了一个类似C函数的宏prologue：所以你可以通过一个调用来使
  用宏：

  myfunc:   prologue 12

  这会把三行代码扩展成如下的样子：
```

myfunc: push ebp mov ebp,esp sub esp,12

在'%macro'一行上宏名后面的数字'1'定义了宏可以接收的参数的个数。 宏定义里面的'%1'是用来引用宏调用中的第一个参数。对于一个有多 个参数的宏，参数序列可以这样写：'%2','%3'等等。

多行宏就像单行宏一样，也是大小写敏感的，除非你使用另一个操作符 ‘%imacro'

如果你必须把一个逗号作为参数的一部分传递给多行宏，你可以把整 个参数放在一个括号中。所以你可以象下面这样编写代码：

```text
  %macro  silly 2

      %2: db      %1

  %endmacro

          silly 'a', letter_a             ; letter_a:  db 'a'
          silly 'ab', string_ab           ; string_ab: db 'ab'
          silly {13,10}, crlf             ; crlf:      db 13,10
```

4.3.1 多行宏的重载

就象单行宏，多行宏也可以通过定义不同的参数个数对同一个宏进行多次 重载。而这次，没有对不带参数的宏的特殊处理了。所以你可以定义：

%macro prologue 0

```text
          push    ebp
          mov     ebp,esp

  %endmacro

  作为函数prologue的另一种形式，它没有开辟本地栈空间。

  有时候，你可能需要重载一个机器指令；比如，你可能想定义：

  %macro  push 2

          push    %1
          push    %2

  %endmacro

  这样，你就可以如下编写代码：

          push    ebx             ; this line is not a macro call
          push    eax,ecx         ; but this one is
```

通常，NASM会对上面的第一行给出一个警告信息，因为'push'现在被定义成 了一个宏，而这行给出的参数个数却不符合宏的定义。但正确的代码还是 会生成的，仅仅是给出一个警告而已。这个警告信息可以通过 '-w'macro-params’命令行选项来禁止。\(参阅2.1.17\)。

4.3.2 Macro-Local Labels

NASM允许你在多行宏中定义labels.使它们对于每一个宏调用来讲是本地的:所 以多次调用同一个宏每次都会使用不同的label.你可以通过在label名称前面 加上'%%'来实现这种用法.所以,你可以创建一条指令,它可以在'Z'标志位被 设置时执行'RET'指令,如下:

```text
  %macro  retz 0

          jnz     %%skip
          ret
      %%skip:

  %endmacro
```

你可以任意多次的调用这个宏,在你每次调用时的时候,NASM都会为'%%skip' 建立一个不同的名字来替换它现有的名字.NASM创建的名字可能是这个样子 的:'..@2345.skip',这里的数字2345在每次宏调用的时候都会被修改.而 '..@'前缀防止macro-local labels干扰本地labels机制,就像在3.9中所 描述的那样.你应该避免在定义你自己的宏时使用这种形式\('..@'前缀,然后 是一个数字,然后是一个句点\),因为它们会和macro-local labels相互产生 干扰.（不理解）

4.3.3 不确定的宏参数个数.

通常,定义一个宏,它可以在接受了前面的几个参数后, 把后面的所有参数都 作为一个参数来使用,这可能是非常有用的,一个相关的例子是,一个宏可能 用来写一个字符串到一个MS-DOS的文本文件中,这里,你可能希望这样写代码:

```text
          writefile [filehandle],"hello, world",13,10
```

NASM允许你把宏的最后一个参数定义成"贪婪参数", 也就是说你调用这个宏时 ,使用了比宏预期得要多得多的参数个数,那所有多出来的参数连同它们之间 的逗号会被作为一个参数传递给宏中定义的最后一个实参,所以,如果你写:

```text
  %macro  writefile 2+

          jmp     %%endstr
    %%str:        db      %2
    %%endstr:
          mov     dx,%%str
          mov     cx,%%endstr-%%str
          mov     bx,%1
          mov     ah,0x40
          int     0x21

   %endmacro
```

那上面使用'writefile'的例子会如预期的那样工作:第一个逗号以前的文本 \[filehandle\]会被作为第一个宏参数使用,会被在'%1'的所有位置上扩展,而 所有剩余的文本都被合并到'%2'中,放在db后面.

这种宏的贪婪特性在NASM中是通过在宏的'%macro'一行上的参数个数后面加 上'+'来实现的.

如果你定义了一个贪婪宏,你就等于告诉NASM对于那些给出超过实际需要的参 数个数的宏调用该如何扩展; 在这种情况下,比如说,NASM现在知道了当它看到 宏调用'writefile'带有2,3或4个或更多的参数的时候,该如何做.当重载宏 时,NASM会计算参数的个数,不允许你定义另一个带有4个参数的'writefile' 宏.

当然,上面的宏也可以作为一个非贪婪宏执行,在这种情况下,调用语句应该 象下面这样写:

```text
            writefile [filehandle], {"hello, world",13,10}
```

NASM提供两种机制实现把逗号放到宏参数中,你可以选择任意一种你喜欢的 形式.

4.3.4 缺省宏参数.

NASM可以让你定义一个多行宏带有一个允许的参数个数范围.如果你这样做了, 你可以为参数指定缺省值.比如:

```text
  %macro  die 0-1 "Painful program death has occurred."

          writefile 2,%1
          mov     ax,0x4c01
          int     0x21

  %endmacro
```

这个宏\(它使用了4.3.3中定义的宏'writefile'\)在被调用的时候可以有一个 错误信息,它会在退出前被显示在错误输出流上,如果它在被调用时不带参数 ,它会使用在宏定义中的缺省错误信息.

通常,你以这种形式指定宏参数个数的最大值与最小值; 最小个数的参数在 宏调用的时候是必须的,然后你要为其他的可选参数指定缺省值.所以,当一 个宏定义以下面的行开始时:

```text
  %macro foobar 1-3 eax,[ebx+2]
```

它在被调用时可以使用一到三个参数, 而'%1'在宏调用的时候必须指定,'%2' 在没有被宏调用指定的时候,会被缺省地赋为'eax','%3'会被缺省地赋为 '\[ebx+2\]'.

你可能在宏定义时漏掉了缺省值的赋值, 在这种情况下,参数的缺省值被赋为 空.这在可带有可变参数个数的宏中非常有用,因为记号'%0'可以让你确定有 多少参数被真正传给了宏.

这种缺省参数机制可以和'贪婪参数'机制结合起来使用;这样上面的'die'宏 可以被做得更强大,更有用,只要把第一行定义改为如下形式即可:

```text
  %macro die 0-1+ "Painful program death has occurred.",13,10
```

最大参数个数可以是无限,以'\*'表示.在这种情况下,当然就不可能提供所有 的缺省参数值. 关于这种用法的例子参见4.3.6.（不理解）

4.3.5 \`%0': 宏参数个数计数器.

对于一个可带有可变个数参数的宏, 参数引用'%0'会返回一个数值常量表示 有多少个参数传给了宏.这可以作为'%rep'的一个参数\(参阅4.5\),以用来遍历 宏的所有参数. 例子在4.3.6中给出.

4.3.6 \`%rotate': 循环移动宏参数.

Unix的shell程序员对于'shift' shell命令再熟悉不过了,它允许把传递给shell 脚本的参数序列\(以'$1,'$2'等引用\)左移一个,所以, 前一个参数是‘$1'的话 左移之后，就变成’$2'可用了，而在'$1'之前是没有可用的参数的。

NASM具有相似的机制，使用'%rotate'。就象这个指令的名字所表达的，它跟Unix 的'shift'是不同的，它不会让任何一个参数丢失，当一个参数被移到最左边的 时候，再移动它，它就会跳到右边。

'%rotate'以单个数值作为参数进行调用\(也可以是一个表达式\)。宏参数被循环 左移，左移的次数正好是这个数字所指定的。如果'%rotate'的参数是负数，那么 宏参数就会被循环右移。

所以，一对用来保存和恢复寄存器值的宏可以这样写：

```text
  %macro  multipush 1-*

    %rep  %0
          push    %1
    %rotate 1
    %endrep

  %endmacro
```

这个宏从左到右为它的每一个参数都依次调用指令'PUSH'。它开始先把它的 第一个参数'%1'压栈，然后调用'%rotate'把所有参数循环左移一个位置，这样 一来，原来的第二个参数现在就可以用'%1'来取用了。重复执行这个过程， 直到所有的参数都被执行完\(这是通过把'%0'作为'%rep'的参数来实现的\)。 这就实现了把每一个参数都依次压栈。

注意，'\*'也可以作为最大参数个数的一个计数，表明你在使用宏'multipush'的 时候，参数个数没有上限。

使用这个宏，确实是非常方便的，执行同等的'POP'操作，我们并不需要把参数 顺序倒一下。一个完美的解决方案是，你再写一个'multipop'宏调用，然后把 上面的调用中的参数复制粘贴过来就行了，这个宏会对所有的寄存器执行相反 顺序的pop操作。

这可以通过下面定义来实现：

```text
  %macro  multipop 1-*

    %rep %0
    %rotate -1
          pop     %1
    %endrep

  %endmacro
```

这个宏开始先把它的参数循环右移一个位置，这样一来，原来的最后一个参数 现在可以用'%1'引用了。然后被pop,然后，参数序列再一次右移，倒数第二个 参数变成了'%1'，就这样，所以参数被以相反的顺序一一被执行。（不理解）

第五章: 汇编器指令。

5.1 \`BITS': 指定目标处理器模式。

'BITS'指令指定NASM产生的代码是被设计运行在16位模式的处理器上还是运行 在32位模式的处理器上。语法是'BITS 16'或'BITS 32'

大多数情况下，你可能不需要显式地指定'BITS'。'aout','coff','elf'和 'win32'目标文件格式都是被设计用在32位操作系统上的，它们会让NASM缺 省选择32位模式。而'obj'目标文件格式允许你为每一个段指定'USE16'或 'USE32'，然后NASM就会按你的指定设定操作模式，所以多次使用'BITS'是 没有必要的。

最有可能使用'BITS'的场合是在一个纯二进制文件中使用32位代码；这是因 为'bin'输出格式在作为DOS的'.COM'程序，DOS的'.SYS'设备驱动程序，或引 导程序时，默认都是16位模式。

如果你仅仅是为了在16位的DOS程序中使用32位指令，你不必指定'BITS 32'， 如果你这样做了，汇编器反而会产生错误的代码，因为这样它会产生运行在 16位模式下，却以32位平台为目标的代码。

当NASM在'BITS 16'状态下时，使用32位数据的指令可以加一个字节的前缀 0x66，要使用32位的地址，可以加上0x67前缀。在'BITS 32'状态下，相反的 情况成立，32位指令不需要前缀，而使用16位数据的指令需要0x66前缀，使 用16位地址的指令需要0x67前缀。

'BITS'指令拥有一个等效的原始形式：\[BITS 16\]和\[BITS 32\]。而用户级的 形式只是一个仅仅调用原始形式的宏。

5.1.1 `USE16' &`USE32': BITS的别名。

'USE16'和'USE32'指令可以用来取代'BITS 16'和'BITS 32',这是为了和其他 汇编器保持兼容性。

5.2 `SECTION'或`SEGMENT': 改变和定义段。

'SECTION'指令\('SEGMENT'跟它完全等效\)改变你正编写的代码将被汇编进的段。 在某些目标文件格式中，段的数量与名称是确定的；而在别一些格式中，用户 可以建立任意多的段。因此，如果你企图切换到一个不存在的段，'SECTION'有 时可能会给出错误信息，或者定义出一个新段，

```text
  Unix的目标文件格式和'bin'目标文件格式，都支持标准的段'.text','.data'
  和'bss'段，与之不同的的，'obj'格式不能辩识上面的段名，并需要把段名开
  头的句点去掉。
```

5.2.1 宏 \`**SECT**'

'SECTION'指令跟一般指令有所不同，的用户级形式跟它的原始形式在功能上有 所不同，原始形式\[SECTION xyz\],简单地切换到给出的目标段。用户级形式， 'SECTION xyz'先定义一个单行宏'**SECT**',定义为原始形式\[SECTION\]，这正 是要执行的指令，然后执行它。所以，用户级指令：

```text
          SECTION .text
```

被展开成两行：

```text
  %define __SECT__        [SECTION .text]
          [SECTION .text]
```

用户会发现在他们自己的宏中，这是非常有用的。比如，4.3.3中定义的宏 'writefile'以下面的更为精致的写法会更有用：

%macro writefile 2+

```text
          [section .data]

    %%str:        db      %2
    %%endstr:

          __SECT__

          mov     dx,%%str
          mov     cx,%%endstr-%%str
          mov     bx,%1
          mov     ah,0x40
          int     0x21

  %endmacro
```

这个形式的宏，一次传递一个用出输出的字符串，先用原始形式的'SECTION'切 换至临时的数据段，这样就不会破会宏'**SECT**'。然后它把它的字符串声明在 数据段中，然后调用'**SECT**'切换加用户先前所在的段。这样就可以避免先前 版本的'writefile'宏中的用来跳过数据的'JMP'指令，而且在一个更为复杂的格 式模型中也不会失败，用户可以把这个宏放在任何独立的代码段中进行汇编。

5.3 \`ABSOLUTE': 定义绝对labels。

'ABSOLUTE'操作符可以被认为是'SECTION'的另一种形式：它会让接下来的代码不 在任何的物理段中，而是在一个从给定地址开始的假想段中。在这种模式中，你 唯一能使用的指令是'RESB'类指令。

```text
  `ABSOLUTE'可以象下面这样使用:

  absolute 0x1A

      kbuf_chr    resw    1
      kbuf_free   resw    1
      kbuf        resw    16
```

这个例子描述了一个关于在段地址0x40处的PC BIOS数据域的段，上面的代码把 'kbuf\_chr'定义在0x1A处，'kbuf\_free'定义在地址0x1C处，'kbuf'定义在地址 0x1E。

就像'SECTION'一样，用户级的'ABSOLUTE'在执行时会重定义'**SECT**'宏。

'STRUC'和'ENDSTRUC'被定义成使用'ABSOLUTE'的宏（同时也使用了'**SECT**'）

'ABSOLUTE'不一定需要带有一个绝对常量作为参数：它也可以带有一个表达式（ 实际上是一个临界表达式，参阅3.8），表达式的值可以是在一个段中。比如，一 个TSR程序可以在用它重用它的设置代码所占的空间：

```text
          org     100h               ; it's a .COM program

          jmp     setup              ; setup code comes last

          ; the resident part of the TSR goes here
  setup:
          ; now write the code that installs the TSR here

  absolute setup

  runtimevar1     resw    1
  runtimevar2     resd    20

  tsr_end:
```

这会在setup段的开始处定义一些变量，所以，在setup运行完后，它所占用的内存 空间可以被作为TSR的数据存储空莘而得到重用。符号'tsr\_end'可以用来计算TSR 程序所需占用空间的大小。

5.4 \`EXTERN': 从其他的模块中导入符中。

'EXTERN'跟MASM的操作符'EXTRN'，C的关键字'extern'极其相似：它被用来声明一 个符号，这个符号在当前模块中没有被定义，但被认为是定义在其他的模块中，但 需要在当前模块中对它引用。不是所有的目标文件格式都支持外部变量的：'bin'文 件格式就不行。

'EXTERN'操作符可以带有任意多个参数，每一个都是一个符号名：

```text
  extern  _printf
  extern  _sscanf,_fscanf
```

有些目标文件格式为'EXTERN'提供了额外的特性。在所有情况下，要使用这些额外 特性，必须在符号名后面加一个冒号，然后跟上目标文件格式相关的一些文字。比如 'obj'文件格式允许你声明一个以外部组'dgroup'为段基址一个变量，可以象下面这样 写：

```text
  extern  _variable:wrt dgroup
```

原始形式的'EXTERN'跟用户级的形式有所不同，因为它只能带有一个参数：对于多个参 数的支持是在预处理器级上的特性。

你可以把同一个变量作为'EXTERN'声明多次：NASM会忽略掉第二次和后来声明的，只采 用第一个。但你不能象声明其他变量一样声明一个'EXTERN'变量。

5.5 \`GLOBAL': 把符号导出到其他模块中。

'GLOBAL'是'EXTERN'的对立面：如果一个模块声明一个'EXTERN'的符号，然后引用它， 然后为了防止链接错误，另外某一个模块必须确实定义了该符号，然后把它声明为 'GLOBAL',有些汇编器使用名字'PUBLIC'。

'GLOBAL'操作符所作用的符号必须在'GLOBAL'之后进行定义。

'GLOBAL'使用跟'EXTERN'相同的语法，除了它所引用的符号必须在同一样模块中已经被 定义过了，比如：

```text
  global _main
  _main:
          ; some code
```

就像'EXTERN'一样，'GLOBAL'允许目标格式文件通过冒号定义它们自己的扩展。比如 'elf'目标文件格式可以让你指定全局数据是函数或数据。

```text
  global  hashlookup:function, hashtable:data
```

就象'EXTERN'一样，原始形式的'GLOBAL'跟用户级的形式不同，仅能一次带有一个参 数

5.6 \`COMMON': 定义通用数据域。

'COMMON'操作符被用来声明通用变量。一个通用变量很象一个在非初始化数据段中定义 的全局变量。所以：

```text
  common  intvar  4

  功能上跟下面的代码相似：

  global  intvar
  section .bss

  intvar  resd    1
```

不同点是如果多于一个的模块定义了相同的通用变量，在链接时，这些通用变量会被 合并，然后，所有模块中的所有的对'intvar'的引用会指向同一片内存。

就角'GLOBAL'和'EXTERN','COMMON'支持目标文件特定的扩展。比如，'obj'文件格式 允许通用变量为NEAR或FAR,而'elf'格式允许你指定通用变量的对齐需要。

```text
  common  commvar  4:near  ; works in OBJ
  common  intarray 100:4   ; works in ELF: 4 byte aligned
```

它的原始形式也只能带有一个参数。

5.7 \`CPU': 定义CPU相关。

'CPU'指令限制只能运行特定CPU类型上的指令。

```text
  选项如下:

  (*) `CPU 8086' 只汇编8086的指令集。

  (*) `CPU 186' 汇编80186及其以下的指令集。

  (*) `CPU 286' 汇编80286及其以下的指令集。

  (*) `CPU 386' 汇编80386及其以下的指令集。

  (*) `CPU 486' 486指令集。

  (*) `CPU 586' Pentium指令集。

  (*) `CPU PENTIUM' 同586。

  (*) `CPU 686' P6指令集。

  (*) `CPU PPRO' 同686

  (*) `CPU P2' 同686

  (*) `CPU P3' Pentium III and Katmai指令集。

  (*) `CPU KATMAI' 同P3

  (*) `CPU P4' Pentium 4 (Willamette)指令集

  (*) `CPU WILLAMETTE' 同P4

  (*) `CPU IA64' IA64 CPU (x86模式下)指令集
```

所有选项都是大小写不敏感的，在指定CPU或更低一级CPU上的所有指令都会 被选择。缺省情况下，所有指令都是可用的。

第六章: 输出文件的格式。

6.1 \`bin': 纯二进制格式输出。

'bin'格式不产生目标文件：除了你编写的那些代码，它不在输出文件中产生 任何东西。这种纯二进制格式的文件可以用在MS-DOS中：'.COM'可执行文件 和'.SYS'设备驱动程序就是纯二进制格式的。纯二进制格式输出对于操作系 统和引导程序开发也是很有用的。

'bin'格式支持多个段名。关于NASM处理'bin'格式中的段的细节，请参阅 6.1.3。

使用'bin'格式会让NASM进入缺省的16位模式 \(参阅5.1\)。为了能在'bin'格 式中使用32位代码，比如在操作系统的内核代码中。你必须显式地使用 'BITS 32'操作符。

'bin'没有缺省的输出文件扩展名：它只是把输入文件的扩展名去掉后作为 输出文件的名字。这样，NASM在缺省模式下会把'binprog.asm'汇编成二进 制文件'binprog'。

6.1.1 \`ORG': 二进制程序的起点位置。

'bin'格式提供一个额外的操作符，这在第五章已经给出'ORG'.'ORG'的功能 是指定程序被载入内存时，它的起始地址。

比如，下面的代码会产生longword: \`0x00000104':

```text
          org     0x100
          dd      label
  label:
```

跟MASM兼容汇编器提供的'ORG'操作符不同，它们允许你在目标文件中跳转， 并覆盖掉你已经产生的代码，而NASM的'ORG'就象它的字面意思“起点”所 表示的，它的功能就是为所有内部的地址引用增加一个段内偏移值；它不允 许MASM版本的'org'的任何其他功能。

6.1.2 `bin'对`SECTION'操作符的扩展。

'bin'输出格式扩展了'SECTION'\(或者'SEGMENT'\)操作符，允许你指定段的 对齐请求。这是通过在段定义行的后面加上'ALIGN'限定符实现的。比如：

```text
  section .data   align=16
```

它切换到段'.data',并指定它必须对齐到16字节边界。

'ALIGN'的参数指定了地址值的低位有多少位必须为零。这个对齐值必须为 2的幂。

6.1.3 \`Multisection' 支持BIN格式.

'bin'格式允许使用多个段，这些段以一些特定的规则进行排列。

```text
  (*) 任何在一个显式的'SECTION'操作符之前的代码都被缺省地加到'.text'
      段中。

  (*) 如果'.text'段中没有给出'ORG'语句，它被缺省地赋为'ORG 0'。

  (*) 显式地或隐式地含有'ORG'语句的段会以'ORG'指定的方式存放。代码
      前会填充0，以在输出文件中满足org指定的偏移。

  (*) 如果一个段内含有多个'ORG'语句，最后一条'ORG'语句会被运用到整
    个段中，不会影响段内多个部分以一定的顺序放到一起。

  (*) 没有'ORG'的段会被放到有'ORG'的段的后面，然后，它们就以第一次声
      明时的顺序被存放。

  (*) '.data'段不像'.text'段和'.bss'段那样，它不遵循任何规则，

  (*) '.bss'段会被放在所有其他段的后面。

  (*) 除非一个更高级别的对齐被指定，所有的段都被对齐到双字边界。

  (*) 段之间不可以交迭。
```

6.2 \`obj': 微软OMF目标文件

'obj'文件格式（因为历史的原因，NASM叫它'obj'而不是'omf'\)是MASM和 TASM可以产生的一种格式，它是标准的提供给16位的DOS链接器用来产生 '.EXE'文件的格式。它也是OS/2使用的格式。

'obj'提供一个缺省的输出文件扩展名'.obj'。

'obj'不是一个专门的16位格式，NASM有一个完整的支持，可以有它的32位 扩展。32位obj格式的文件是专门给Borland的Win32编译器使用的，这个编 译器不使用微软的新的'win32'目标文件格式。

'obj'格式没有定义特定的段名字：你可以把你的段定义成任何你喜欢 的名字。一般的，obj格式的文件中的段名如：`CODE',`DATA'和\`BSS'.

如果你的源文件在显式的'SEGMENT'前包含有代码，NASM会为你创建一个叫 做\`\_\_NASMDEFSEG'的段以包含这些代码.

当你在obj文件中定义了一个段，NASM把段名定义为一个符号，所以你可以 存取这个段的段地址。比如：

```text
  segment data

  dvar:   dw      1234

  segment code

  function:
          mov     ax,data         ; get segment address of data
          mov     ds,ax           ; and move it into DS
          inc     word [dvar]     ; now this reference will work
          ret
```

obj格式中也可以使用'SEG'和'WRT'操作符，所以你可以象下面这样编写代码：

```text
  extern  foo

        mov   ax,seg foo            ; get preferred segment of foo
        mov   ds,ax
        mov   ax,data               ; a different segment
        mov   es,ax
        mov   ax,[ds:foo]           ; this accesses `foo'
        mov   [es:foo wrt data],bx  ; so does this
```

6.2.1 `obj' 对`SEGMENT'操作符的扩展。

obj输出格式扩展了'SEGMENT'\(或'SECTION'\)操作符，允许你指定段的多个 属性。这是通过在段定义行的末尾添加额外的限定符来实现的，比如：

```text
  segment code private align=16
```

这定义了一个段'code'，但同时把它声明为一个私有段，同时，它所描述的 这个部分必须被对齐到16字节边界。

```text
  可用的限定符如下:

  (*) `PRIVATE', `PUBLIC', `COMMON'和`STACK' 指定段的联合特征。`PRIVATE'
    段在连接时不和其他的段进行连接；'PUBLIC'和'STACK'段都会在连接时
    连接到一块儿；而'COMMON‘段都会在同一个地址相互覆盖，而不会一接一
    个连接好。

  (*) 就象上面所描述的，'ALIGN'是用来指定段基址的低位有多少位必须为零，
    对齐的值必须以2的乘方的形式给出，从1到4096；实际上，真正被支持的
    值只有1，2，4，16，256和4096，所以如果你指定了8，它会自动向上对齐
    到16，32，64会对齐到128等等。注意，对齐到4096字节的边界是这种格式
    的PharLap扩展，可能所有的连接器都不支持。

  (*) 'CLASS'可以用来指定段的类型；这个特性告诉连接器，具有相同class的
    段应该在输出文件中被放到相近的地址。class的名字可以是任何字。比如
    'CLASS=CODE'。

  (*) 就象`CLASS', `OVERLAY'通过一个作为参数的字来指定，为那些有覆盖能力
      的连接器提供覆盖信息。

  (*) 段可以被声明为'USE16'或'USE32'，这种选择会对目标文件产生影响，同时
      在段内16位或32位代码分开的时候，也能保证NASM的缺省汇编模式

  (*) 当编写OS/2目标文件的时候，你应当把32位的段声明为'FLAT'，它会使缺省
     的段基址进入一个特殊的组'FLAT'，同时，在这个组不存在的时候，定义这
     个组。

  (*) obj文件格式也允许段在声明的时候，前面有一个定义的绝对段地址，尽管没
      有连接器知道这个特性应该怎么使用；但如果你需要的话，NASM还是允许你
      声明一个段如下面形式：`SEGMENT SCREEN ABSOLUTE=0xB800'`ABSOLUTE'和
      `ALIGN'关键字是互斥的。
```

NASM的缺省段属性是`PUBLIC',`ALIGN=1', 没有class,没有覆盖, 并 \`USE16'.

6.2.2 \`GROUP': 定义段组。

obj格式也允许段被分组，所以一个单独的段寄存器可以被用来引用一个组中的 所有段。NASM因此提供了'GROUP'操作符，据此，你可以这样写代码：

```text
  segment data

          ; some data

  segment bss

          ; some uninitialised data

  group dgroup data bss
```

这会定义一个叫做'dgroup'的组，包含有段'data'和'bss'。就象'SEGMENT', 'GROUP'会把组名定义为一个符号，所以你可以使用'var wrt data'或者'var wrt dgroup'来引用'data'段中的变量'var'，具体用哪一个取决于哪一个段 值在你的当前段寄存器中。

如果你只是想引用'var',同时，'var'被声明在一个段中，段本身是作为一个 组的一部分，然后，NASM缺省给你的'var'的偏移值是从组的基地址开始的， 而不是段基址。所以，'SEG var'会返回组基址而不是段基址。

NASM也允许一个段同时作为多个组的一个部分，但如果你真这样做了，会产生 一个警告信息。段内同时属于多个组的那些变量在缺省状况下会属于第一个被 声明的包含它的组。

一个组也不一定要包含有段；你还是可以使用'WRT'引用一个不在组中的变量。 比如说，OS/2定义了一个特殊的组'FLAT'，它不包含段。

6.2.3 \`UPPERCASE': 在输出文件中使大小写敏感无效。

尽管NASM自己是大小写敏感的，有些OMF连接器并不大小写敏感；所以，如果 NASM能输出大小写单一的目标文件会很有用。'UPPERCASE'操作符让所有的写 入到目标文件中的组，段，符号名全部强制为大写。在一个源文件中，NASM 还是大小写敏感的；但目标文件可以按要求被整个写成是大写的。

'UPPERCASE'写在单独一行中，不需要任何参数。

6.2.4 \`IMPORT': 导入DLL符号。

如果你正在用NASM写一个DLL导入库,'IMPORT'操作符可以定义一个从DLL库中 导入的符号,你使用'IMPORT'操作符的时候,你仍旧需要把符号声明为'EXTERN'.

'IMPORT'操作符需要两个参数,以空格分隔,它们分别是你希望导入的符号的名 称和你希望导入的符号所在的库的名称,比如:

```text
      import  WSAStartup wsock32.dll
```

第三个参数是可选的,它是符号在你希望从中导入的链接库中的名字,这样的话, 你导入到你的代码中的符号可以和库中的符号不同名,比如:

```text
      import  asyncsel wsock32.dll WSAAsyncSelect
```

6.2.5 \`EXPORT': 导出DLL符号.

'EXPORT'也是一个目标格式相关的操作符,它定义一个全局符号,这个符号可以被 作为一个DLL符号被导出,如果你用NASM写一个DLL库.你可以使用这个操作符,在 使用中,你仍旧需要把符号定义为'GLOBAL'.

'EXPORT'带有一个参数,它是你希望导出的在源文件中定义的符号的名字.第二个 参数是可选的\(跟第一个这间以空格分隔\),它给出符号的外部名字,即你希望让使 用这个DLL的应用程序引用这个符号时所用的名字.如果这个名字跟内部名字同名, 可以不使用第二个参数.

还有一些附加的参数,可以用来定义导出符号的一些属性.就像第二个参数, 这些 参数也是以空格分隔.如果要给出这些参数,那么外部名字也必须被指定,即使它跟 内部名字相同也不能省略,可用的属性如下:

```text
  (*) 'resident'表示某个导出符号在系统引导后一直常驻内存.这对于一些经常使用
      的导出符号来说,是很有用的.

  (*) `nodata'表示导出符号是一个函数,这个函数不使用任何已经初始化过的数据.

  (*) `parm=NNN', 这里'NNN'是一个整型数,当符号是一个在32位段与16位段之间的
    调用门时,它用来设置参数的尺寸大小(占用多少个wrod).

  (*) 还有一个属性,它仅仅是一个数字,表示符号被导出时带有一个标识数字.

  比如:

      export  myfunc
      export  myfunc TheRealMoreformalLookingFunctionName
      export  myfunc myfunc 1234  ; export by ordinal
      export  myfunc myfunc resident parm=23 nodata
```

6.2.6 \`..start': 定义程序的入口点.

'OMF'链接器要求被链接进来的所有目标文件中,必须有且只能有一个程序入口点, 当程序被运行时,就从这个入口点开始.如果定义这个入口点的目标文件是用 NASM汇编的,你可以通过在你希望的地方声明符号'..start'来指定入口点.

6.2.7 `obj'对`EXTERN'操作符的扩展.

如果你以下面的方式声明了一个外部符号:

```text
      extern  foo
```

然后以这样的方式引用'mov ax,foo',这样只会得到一个关于foo的偏移地址,而且 这个偏移地址是以'foo'的首选段基址为参考的\(在'foo'被定义的这个模块中指定 的段\).所以,为了存取'foo'的内容,你实际上需要这样做:

```text
          mov     ax,seg foo      ; get preferred segment base
          mov     es,ax           ; move it into ES
          mov     ax,[es:foo]     ; and use offset `foo' from it
```

这种方式显得稍稍有点笨拙,实际上如果你知道一个外部符号可以通过给定的段 或组来进行存的话,假定组'dgroup'已经在DS寄存器中,你可以这样写代码:

```text
          mov     ax,[foo wrt dgroup]
```

但是,如果你每次要存取'foo'的时候,都要打这么多字是一件很痛苦的事情;所以 NASM允许你声明'foo'的另一种形式:

```text
      extern  foo:wrt dgroup
```

这种形式让NASM假定'foo'的首选段基址是'dgroup';所以,表达式'seg foo'现在会 返回'dgroup',表达式'foo'等同于'foo wrt dgroup'.

缺省的'WRT'机制可以用来让外部符号跟你程序中的任何段或组相关联.他也可以被 运用到通用变量上,参阅6.2.8.

6.2.8 `obj'对`COMMON'操作符的扩展.

'obj'格式允许通用变量为near或far;NASM允许你指定你的变量属于哪一类,语法如 下:

```text
  common  nearvar 2:near   ; `nearvar' is a near common
  common  farvar  10:far   ; and `farvar' is far
```

Far通用变量可能会大于64Kb,所以OMF可以把它们声明为一定数量的指定的大小的元 素.比如,10byte的far通用变量可以被声明为10个1byte的元素,5个2byte的元素,或 2个5byte的元素,或1个10byte的元素.

有些'OMF'链接器需要元素的size,同时需要变量的size,当在多个模块中声明通用变 量时可以用来进行匹配.所以NASM必须允许你在你的far通用变量中指定元素的size. 这可以通过下面的语法实现:

```text
  common  c_5by2  10:far 5        ; two five-byte elements
  common  c_2by5  10:far 2        ; five two-byte elements
```

如果元素的size没有被指定,缺省值是1.还有,如果元素size被指定了,那么'far'关键 字就不需要了,因为只有far通用变量是有元素size的.所以上面的声明等同于:

```text
  common  c_5by2  10:5            ; two five-byte elements
  common  c_2by5  10:2            ; five two-byte elements
```

这种扩展的特性还有,'obj'中的'COMMON'操作符还可以象'EXTERN'那样支持缺省的 'WRT'指定,你也可以这样声明:

```text
  common  foo     10:wrt dgroup
  common  bar     16:far 2:wrt data
  common  baz     24:wrt data:6
```

6.3 \`win32': 微软Win32目标文件

'win32'输出格式产生微软win32目标文件,可以用来给微软连接器进行连接,比如 Visual C++.注意Borland Win32编译器不使用这种格式,而是使用'obj'格式\(参阅 6.2\)

'win32'提供缺省的输出文件扩展名'.obj'.

注意,尽管微软声称Win32目标文件遵循'COFF'标准\(通用目标文件格式\),但是微软 的Win32编译器产生的目标文件和一些COFF连接器\(比如DJGPP\)并不兼容,反过来也 一样.这是由一些PC相关的语义上的差异造成的. 使用NASM的'coff'输出格式,可以 产生能让DJGPP使用的COFF文件; 而这种'coff'格式不能产生能让Win32连接器正确 使用的代码.

6.3.1 `win32'对`SECTION'的扩展.

就象'obj'格式,'win32'允许你在'SECTION'操作符的行上指定附加的信息,以用来控 制你声明的段的类型与属性.对于标准的段名'.text','.data',和'.bss',类型和属 性是由NASM自动产生的,但是还是可以通过一些限定符来重新指定:

```text
  可用的限定符如下:

  (*) 'code'或者'text',把一个段定义为一个代码段,这让这个段可读并可执行,但是
      不能写,同时也告诉连接器,段的类型是代码段.

  (*) 'data'和'bss'定义一个数据段,类似'code',数据段被标识为可读可写,但不可执
      行,'data'定义一个被初始化过的数据段,'bss'定义一个未初始化的数据段.

  (*) 'rdata'声明一个初始化的数据段,它可读,但不能写.微软的编译器把它作为一
      个存放常量的地方.

  (*) 'info'定义一个信息段,它不会被连接器放到可执行文件中去,但可以传递一些
      信息给连接器.比如,定义一个叫做'.drectve'信息段会让连接器把这个段内的
      内容解释为命令行选项.

  (*) 'align='跟上一个数字,就象在'obj'格式中一样,给出段的对齐请求.你最大可
      以指定64:Win32目标文件格式没有更大的段对齐值.如果对齐请求没有被显式
      指定,缺省情况下,对于代码段,是16byte对齐,对于只读数据段,是8byte对齐,对
      于数据段,是4byte对齐.而信息段缺省对齐是1byte(即没有对齐),所以对它来说,
      指定的数值没用.
```

如果你没有指定上述的限定符,NASM的缺省假设是:

```text
  section .text    code  align=16
  section .data    data  align=4
  section .rdata   rdata align=8
  section .bss     bss   align=4
```

任何其的段名都会跟'.text'一样被对待.

6.4 \`coff': 通用目标文件格式.

'coff'输出类型产生'COFF'目标文件,可以被DJGPP用来连接.

'coff'提供一个缺省的输出文件扩展名'.o'.

'coff'格式支持跟'win32'同样的对于'SECTION'的扩展,除了'align'和'info'限 定符不被支持.

6.5.1 `elf'对`SECTION'操作符的扩展.

就象'obj'格式一样,'elf'允许你在'SECTION'操作符行上指定附加的信息,以控制 你声明的段的类型与属性.对于标准的段名'.text','.data','.bss',NASM都会产 生缺省的类型与属性.但还是可以通过一些限定符与重新指定.

可用的限定符如下:

```text
  (*) 'alloc'定义一个段,在程序运行时,这个段必须被载入内存中,'noalloc'正好
      相反,比如信息段,或注释段.

  (*) 'exec'把段定义为在程序运行的时候必须有执行权限.'noexec'正好相反.

  (*) `write'把段定义为在程序运行时必须可写,'nowrite'正好相反.

  (*) `progbits'把段定义为在目标文件中必须有实际的内容,比如象普通的代码段
      与数据段,'nobits'正好相反,比如'bss'段.

  (*) `align='跟上一个数字,给出段的对齐请求.

  如果你没有指定上述的限定符信息,NASM缺省指定的如下:

  section .text    progbits  alloc  exec    nowrite  align=16
  section .rodata  progbits  alloc  noexec  nowrite  align=4
  section .data    progbits  alloc  noexec  write    align=4
  section .bss     nobits    alloc  noexec  write    align=4
  section other    progbits  alloc  noexec  nowrite  align=1
```

\(任何不在上述列举范围内的段,在缺省状况下,都被作为'other'段看待\).

6.5.2 地址无关代码: `elf'特定的符号和`WRT'

'ELF'规范含有足够的特性以允许写地址无关\(PIC\)的代码,这可以让ELF非常 方便地共享库.尽管如此,这也意味着NASM如果想要成为一个能够写PIC的汇 编器的话,必须能够在ELF目标文件中产生各种奇怪的重定位信息,

因为'ELF'不支持基于段的地址引用,'WRT'操作符不象它的常规方式那样被 使用,所以,NASM的'elf'输出格式中,对于'WRT'有特殊的使用目的,叫做: PIC相关的重定位类型.

'elf'定义五个特殊的符号,它们可以被放在'WRT'操作符的右边用来实现PIC 重定位类型.它们是`..gotpc',`..gotoff', `..got',`..plt' and \`..sym'. 它们的功能简要介绍如下:

```text
  (*) 使用'wrt ..gotpc'来引用以global offset table为基址的符号会得到
      当前段的起始地址到global offset table的距离.(
      `_GLOBAL_OFFSET_TABLE_'是引用GOT的标准符号名).所以你需要在返回
      结果前面加上'$$'来得到GOT的真实地址.

  (*) 用'wrt ..gotoff'来得到你的某一个段中的一个地址实际上得到从GOT的
    的起始地址到你指定的地址之间的距离,所以这个值再加上GOT的地址为得
    到你需要的那个真实地址.

  (*) 使用'wrt ..got'来得到一个外部符号或全局符号会让连接器在含有这个
      符号的地址的GOT中建立一个入口,这个引用会给出从GOT的起始地址到这
      个入口的一个距离;所以你可以加上GOT的地址,然后从得到的地址处载入,
      就会得到这个符号的真实地址.

  (*) 使用'wrt ..plt'来引用一个过程名会让连接器建立一个过程连接表入口,
      这个引用会给出PLT入口的地址.你可以在上下文使用这个引用,它会产生
      PC相关的重定位信息,所以,ELF包含引用PLT入口的非重定位类型

  (*) 略
```

在8.2章中会有一个更详细的关于如何使用这些重定位类型写共享库的介绍

6.5.3 `elf'对`GLOBAL'操作符的扩展.

'ELF'目标文件可以包含关于一个全局符号的很多信息,不仅仅是一个地址:他 可以包含符号的size,和它的类型.这不仅仅是为了调试的方便,而且在写共享 库程序的时候,这确实是非常有用的.所以,NASM支持一些关于'GLOBAL'操作符 的扩展,允许你指定这些特性.

你可以把一个全局符号指定为一个函数或一个数据对象,这是通过在名字后面 加上一个冒号跟上'function'或'data'实现的.\('object'可以用来代替'data'\) 比如:

```text
  global   hashlookup:function, hashtable:data
```

把全局符号'hashlookup'指定为一个函数,把'hashtable'指定为一个数据对象.

你也可以指定跟这个符号关联的数据的size,可以一个数值表达式\(它可以包含 labels,甚至前向引用\)跟在类型后面,比如:

```text
  global  hashtable:data (hashtable.end - hashtable)

  hashtable:
          db this,that,theother  ; some data here
  .end:
```

这让NASM自动计算表的长度,然后把信息放进'ELF'的符号表中.

声明全局符号的类型和size在写共享库代码的时候是必须的,关于这方面的更多 信息,参阅8.2.4.

6.5.4 `elf'对`COMMON'操作符的扩展.

'ELF'也允许你指定通用变量的对齐请求.这是通过在通用变量的名字和size的 后面加上一个以冒号分隔的数字来实现的,比如,一个doubleword的数组以 4byte对齐比较好:

```text
  common  dwordarray 128:4
```

这把array总的size声明为128bytes,并确定它对齐到4byte边界.

6.5.5 16位代码和ELF

'ELF32'规格不提供关于8位和16位值的重定位,但GNU的连接器'ld'把这作为 一个扩展加进去了.NASM可以产生GNU兼容的重定位,允许16位代码被'ld'以 'ELF'格式进行连接.如果NASM使用了选项'-w+gnu-elf-extensions',如果一 个重定位被产生的话,会有一条警告信息.

6.6 `aout': Linux`a.out' 目标文件

'aout'格式产生'a.out'目标文件,这种格式在早期的Linux系统中使用\(现在的 Linux系统一般使用ELF格式,参阅6.5\),这种格式跟其他的'a.out'目标文件有 所不同,文件的头四个字节的魔数不一样;还有,有些版本的'a.out',比如NetBSD 的,支持地址无关代码,这一点,Linux的不支持.

'a.out'提供的缺省文件扩展名是'.o'.

'a.out'是一种非常简单的目标文件格式.它不支持任何特殊的操作符,没有特殊 的符号,不使用'SEG'或'WRT',对于标准的操作符也没有任何扩展.它只支持三个 标准的段名'.text','.data','.bss'.

6.7 `aoutb': NetBSD/FreeBSD/OpenBSD`a.out'目标文件.

'aoutb'格式产生在BSD unix,NetBSD,FreeBSD,OpenBSD系统上使用的'a.out'目 标文件. 作为一种简单的目标文件,这种格式跟'aout'除了开头四字节的魔数不 一样,其他完全相同.但是,'aoutb'格式支持跟elf格式一样的地址无关代码,所以 你可以使用它来写'BSD'共享库.

'aoutb'提供的缺省文件扩展名是'.o'.

'aoutb'不支持特殊的操作符,没有特殊的符号,只有三个殊殊的段名'.text', '.data'和'.bss'.但是,它象elf一样支持'WRT'的使用,这是为了提供地址无关的 代码重定位类型.关于这部分的完整文档,请参阅6.5.2

'aoutb'也支持跟'elf'同样的对于'GLOBAL'的扩展:详细信息请参阅6.5.3.

6.8 `as86': Minix/Linux`as86'目标文件.

Minix/Linux 16位汇编器'as86'有它自己的非标准目标文件格式. 虽然它的链 接器'ld86'产生跟普通的'a.out'非常相似的二进制输出,在'as86'跟'ld86'之 间使用的目标文件格式并不是'a.out'.

NASM支持这种格式,因为它是有用的,'as86'提供的缺省的输出文件扩展名是'.o'

'as86'是一个非常简单的目标格式\(从NASM用户的角度来看\).它不支持任何特殊 的操作符,符号,不使用'SEG'或'WRT',对所有的标准操作符也没有任何扩展.它只 支持三个标准的段名:'.text','.data',和'.bss'.

6.9 \`rdf': 可重定位的动态目标文件格式.

'rdf'输出格式产生'RDOFF'目标文件.'RDOFF'\(可重定位的动态目标文件格式\) \`RDOFF'是NASM自产的目标文件格式,是NASM自已设计的,它被反映在汇编器的内 部结构中.

'RDOFF'在所有知名的操作系统中都没有得到应用.但是,那些正在写他们自己的 操作系统的人可能非常希望使用'RDOFF'作为他们自己的目标文件格式,因为 'RDOFF'被设计得非常简单,并含有很少的冗余文件头信息.

NASM的含有源代码的Unix包和DOS包中都含有一个'rdoff'子目录,里面有一套 RDOFF工具:一个RDF连接器,一个RDF静态库管理器,一个RDF文件dump工具,还有 一个程序可以用来在Linux下载入和执行RDF程序.

'rdf'只支持标准的段名'.text','.data','.bss'.

6.9.1 需要一个库: \`LIBRARY'操作符.

'RDOFF'拥有一种机制,让一个目标文件请求一个指定的库被连接进模块中,可以 是在载入时,也可以是在运行时连接进来.这是通过'LIBRARY'操作符完成的,它带 有一个参数,即这个库的名字:

```text
      library  mylib.rdl
```

6.9.2 指定一个模块名称: \`MODULE'操作符.

特定的'RDOFF'头记录被用来存储模块的名字.它可以被用在运行时载入器作动 态连接.'MODULE'操作符带有一个参数,即当前模块的名字:

```text
      module  mymodname
```

注意,当你静态连接一个模块,并告诉连接器从输出文件中除去符号时,所有的模 块名字也会被除去.为了避免这种情况,你应当在模块的名字前加一个'$',就像:

```text
      module  $kernel.core
```

6.9.3 `rdf'对`GLOBAL'操作符的扩展.

'RDOFF'全局符号可以包含静态连接器需要的额外信息.你可以把一个全局符号 标识为导出的,这就告诉连接器不要把它从目标可执行文件中或库文件中除去. 就象在'ELF'中一样,你也可以指定一个导出符号是一个过程或是一个数据对象.

在名字的尾部加上一个冒号和'exporg',你就可以让一个符号被导出:

```text
      global  sys_open:export
```

要指定一个导出符号是一个过程\(函数\),你要在声明的后南加上'proc'或'function'

```text
      global  sys_open:export proc
```

相似的,要指定一个导出的数据对象,把'data'或'object'加到操作符的后面:

```text
      global  kernel_ticks:export data
```

6.10 \`dbg': 调试格式.

在缺省配置下,'dbg'输出格式不会被构建进NASM中.如果你是从源代码开始构建你 自己的NASM可执行版本,你可以在'outform.h'中定义'OF\_DBG'或在编译器的命令 行上定义,这样就可以得到'dbg'输出格式.

'dbg'格式不输出一个普通的目标文件;它输出一个文本文件,包含有一个关于到输 出格式的最终模块的转化动作的列表.它主要是用于帮助那些希望写自己的驱动程 序的用户,这样他们就可以得到一个关于主程序的各种请求在输出中的形式的完整 印象.

对于简单的文件,可以简单地象下面这样使用:

```text
  nasm -f dbg filename.asm
```

这会产生一个叫做'filename.dgb'的诊断文件.但是,这在另一些目标文件上可能工 作得并不是很好,因为每一个目标文件定义了它自己的宏\(通常是用户级形式的操作 符\),而这些宏在'dbg'格式中并没有定义.因此,运行NASM两遍是非常有用的,这是为 了对选定的源目标文件作一个预处理:

```text
  nasm -e -f rdf -o rdfprog.i rdfprog.asm
  nasm -a -f dbg rdfprog.i
```

这先把'rdfprog.asm先预处理成'rdfprog.i',让RDF特定的操作符被正确的转化成 原始形式.然后,被预处理过的源程序被交给'dbg'格式去产生最终的诊断输出.

这种方式对于'obj'格式还是不能正确工作的,因为'obj'的'SEGMENT'和'GROUP'操 作符在把段名与组名定义为符号的时候会有副作用;所以程序不会被汇编.如果你 确实需要trace一个obj的源文件,你必须自己定义符号\(比如使用'EXTERN'\)

'dbg'接受所有的段名与操作符,并把它们全部记录在自己的输出文件中.

### 第七章: 编写16位代码 \(DOS, Windows 3/3.1\)

本章将介绍一些在编写运行在'MS-DOS'和'Windows 3.x'下的16位代码的时候需要 用到的一些常见的知识.涵兽了如果连接程序以生成.exe或.com文件,如果编写 .sys设备驱动程序,以及16位的汇编语言代码与C编译器和Borland Pascal编译器 之间的编程接口.

7.1 产生'.EXE'文件.

DOS下的任何大的程序都必须被构建成'.EXE'文件,因为只有'.EXE'文件拥有一种 内部结构可以突破64K的段限制.Windows程序也需要被构建成'.EXE'文件,因为 Windows不支持'.COM'格式.

一般的,你是通过使用一个或多个'obj'格式的'.OBJ'目标文件来产生'.EXE'文件 的,用连接器把它们连接到一起.但是,NASM也支持通过'bin'输出格式直接产生一 个简单的DOS '.EXE'文件\(通过使用'DB'和'DW'来构建exe文件头\),并提供了一组 宏帮助做到这一点.多谢Yann Guidon贡献了这一部分代码.

在NASM的未来版本中,可能会完全支持'.EXE'文件.

7.1.1 使用'obj'格式来产生'.EXE'文件.

本章选描述常见的产生'.EXE'文件的方法:把'.OBJ'文件连接到一起.

大多数16位的程序语言包都附带有一个配套的连接器,如果你没有,有一个免费的 叫做VAL的连接器,在`x2ftp.oulu.fi'上可以以'LZH'包的格式得到.也可以在`ftp.simtel.net'上得到. 另一个免费的LZH包\(尽管这个包是没有源代码的\),叫做 FREELINK,可以在`www.pcorner.com'上得到. 第三个是'djlink',是由DJ Delorie写 的,可以在`www.delorie.com'上得到. 第四个 'ALINK', 是由Anthony A.J. Williams 写的,可以在\`alink.sourceforge.net'上得到.

当把多个'.OBJ'连接进一个'.EXE'文件中的时候,你需要保证它们当中有且仅有一 个含有程序入口点\(使用'obj'格式定义的特殊符号'..start'参阅6.2.6\).如果没有 模块定义入口点,连接器就不知道在输出文件的文件头中为入口点域赋什么值,如 果有多个入口被定义,连接器就不知道到底该用哪一个.

一个关于把NASM源文件汇编成'.OBJ'文件,并把它连接成一个'.EXE'文件的例子在 这里给出.它演示了定义栈,初始化段寄存器,声明入口点的基本做法.这个文件也 在NASM的'test'子目录中有提供,名字是'objexe.asm'.

```text
  segment code

  ..start:
          mov     ax,data
          mov     ds,ax
          mov     ax,stack
          mov     ss,ax
          mov     sp,stacktop
```

这是一段初始化代码，先把DS寄存器设置成指定数据段，然后把‘SS’和‘SP’寄存器 设置成指定提供的栈。注意，这种情况下，在'mov ss,ax'后，有一条指令隐式地把 中断关闭掉了，这样抗敌，在载入 'SS'和‘SP’的过程中就不会有中断发生，并且没 有可执行的栈可用。

还有，一个特殊的符号'..start'在这段代码的开头被定义，它表示最终可执行代 码的入口点。

```text
          mov     dx,hello
          mov     ah,9
          int     0x21
```

上面是主程序：在'DS:DX'中载入一个指向欢迎信息的指针\('hello'隐式的跟段 ‘data'相关联，’data'在设置代码中已经被载入到‘DS‘寄存器中，所以整个指针是 有效的\)，然后调用DOS的打印字符串功能调用。

```text
          mov     ax,0x4c00
          int     0x21
```

这两句使用另一个DOS功能调用结束程序。

```text
  segment data

  hello:  db      'hello, world', 13, 10, '$'

  数据段中含有我们想要显示的字符串。

  segment stack stack
          resb 64
  stacktop:

  上面的代码声明一个含有64bytes的未初始化栈空间的堆栈段，然后把指针
  ’stacktop'指向它的顶端。操作符'segment stack stack'定义了一个叫做
  ‘stack'的段，同时它的类型也是'STACK'.后者并不一定需要，但是连接串可
  能会因为你的程序中没有段的类型为'STACK'而发出警告。

  上面的文件在被编译为'.OBJ'文件中，会自动连接成为一个有效的'.EXE'文
  件，当运行它时会打印出'hello world'，然后退出。
```

7.1.2 使用’bin'格式来产生\`.EXE'文件。

'.EXE'文件是相当简单的，所以可以通过编写一个纯二进制文件然后在前面 连接上一个32bytes的头就可以产生一个'.exe'的文件了。这个文件头也是 相当简单，它可以通过使用NASM自己的'DB'和'DW'命令来产生，所以你可以使 用'bin'输出格式直接产生'.EXE'文件。

在NASM的包中，有一个'misc'子目录，这是一个宏文件'exebin.mac'。它定义 了三个宏`EXE_begin',`EXE\_stack'和\`EXE\_end'.

要通过这种方法产生一个'.EXE'文件，你应当开始的时候先使用'%include'载 入'exebin.mac'宏包到你的源文件中。然后，你应当使用'EXE\_begin'宏\(不带 任何参数\)来产生文件头数据。然后像平常一样写二进制格式的代码-你可以 使用三种标准的段'.text','.data','.bss'.在文件的最后，你应当调用 'EXE\_end'宏\(还是不带任何参数\)，它定义了一些标识段size的符号，而这些宏 会由'EXE\_begin'产生的文件头代码引用。

在这个模块中，你最后的代码是写在'0x100'开始的地址处的，就像是'.COM'文 件-实际上，如果你剥去那个32bytes的文件头，你就会得到一个有效的'.COM'程 序。所有的段基址是相同的，所以程序的大小被限制在64K的范围内，这还是跟 一个'.COM'文件相同。'ORG'操作符是被'EXE\_begin'宏使用的，所以你不必自己 显式的使用它

你可以直接使用你的段基址，但不幸的是，因为这需要在文件头中有一个重定 位，事情就会变得更复杂。所以你应当从'CS'中拷贝出一个段基址。

进入你的'.EXE'文件后，'SS:SP'已经被正确的指向一个2Kb的栈顶。你可以通过 调用'EXE\_stack'宏来调整缺省的2KB的栈大小。比如，把你的栈size改变到 64bytes,你可以调用'EXE\_stack 64'

一个关于以这种方式产生一个'.EXE'文件的例子在NASM包的子目录'test'中, 名字是'binexe.asm'

7.2 产生\`.COM'文件

一个大的DOS程序最好是写成'.EXE'文件，但一个小的程序往往最好写成'.COM' 文件。'.COM'文件是纯二进制的，所以使用'bin'输出格式可以很容易的地产生。

7.2.1 使用`bin'格式产生`.COM’文件。

'.COM'文件预期被装载到它们所在段的'100h'偏移处\(尽管段可能会变\)。然后 从100h处开始执行，所以要写一个'.COM'程序，你应当象下面这样写代码：

```text
          org 100h

  section .text

  start:
          ; put your code here

  section .data

          ; put data items here

  section .bss

          ; put uninitialised data here
```

'bin'格式会把'.text'段放在文件的最开始处，所以如果你需要，你可以在开始 编写代码前先声明data和bss元素，代码段最终还是会放到文件的最开始处。

BSS\(未初始化过的数据\)段本身在'.COM'文件中并不占据空间：BSS中的元素的地 址是一个指向文件外面的一个空间的一个指针，这样做的依据是在程序运行中， 这样可以节省空间。所以你不应当相信当你运行程序时，你的BSS段已经被初始 化为零了。

为了汇编上面的程序，你应当象下面这样使用命令行：

```text
  nasm myprog.asm -fbin -o myprog.com

  如果没有显式的指定输出文件名，这个'bin'格式会产生一个叫做'myprog'的文
  件，所以你必须重新给它指定一个文件名。
```

7.2.2 使用`obj'格式产生`.COM'文件

如果你在写一个'.COM'文件的时候，产生了多于一个的模块，你可能希望汇编成 多个'.OBJ'文件，然后把它们连接成一个'.COM'程序。如果你拥有一个能够输出 '.COM'文件的连接器，你可以做到这一点。或者拥有一个转化程序\(比如， 'EXE2BIN'\)把一个'.EXE'输出文件转化为一个'.COM'文件也可。

如果你要这样做，你必须注意几件事情：

```text
  (*) 第一个含有代码的目标文件在它的代码段中，第一句必须是：'RESB 100h'。
      这是为了保证代码在代码段基址的偏移'100h'处开始，这样，连接器和转化
```

程序在产生.com文件时，就不必调整地址引用了。其他的汇编器是使用'ORG' 操作符来达到此目的的，但是'ORG'在NASM中对于'bin'格式来说是一个格式相 关的操作符，会表达不同的含义。

```text
  (*) 你不必定义一个堆栈段。

  (*) 你的所有段必须放在一个组中，这样每次你的代码或数据引用一个符号偏移
      时，所有的偏移值都是相对于同一个段基址的。这是因为，当一个'.COM'文件
```

载入时，所有的段寄存器含有同一个值。

第八章: 编写32位代码\(Unix, Win32, DJGPP\)

第九章: 混合16位与32位代码

汇编语言中，串操作指令LODSB/LODSW是块装入指令，其具体操作是把SI指向的存储单元读入累加器,LODSB就读入AL,LODSW就读入AX中,然后SI自动增加或减小1或2.其常常是对数组或字符串中的元素逐个进行处理。

### 汇编 db,dw,dd的区别

db定义字节类型变量，一个字节数据占1个字节单元，读完一个，偏移量加1

dw定义字类型变量，一个字数据占2个字节单元，读完一个，偏移量加2

dd定义双字类型变量，一个双字数据占4个字节单元，读完一个，偏移量加4

### ALIGN

ALIGN 伪指令将一个变量对齐到字节边界、字边界、双字边界或段落边界。

为了满足对齐要求，汇编器会在变量前插入一个或多个空字节。为什么要对齐数据？因为，对于存储于偶地址和奇地址的数据来说，CPU 处理偶地址数据的速度要快得多。

对齐伪指令ALIGN
对齐伪指令格式：

ALIGN Num

其中：Num必须是2的幂，如：2、4、8和16等。

伪指令的作用是：告诉汇编程序，本伪指令下面的内存变量必须从下一个能被Num整除的地址开始分配。

如果下一个地址正好能被Num整除，那么，该伪指令不起作用，否则，汇编程序将空出若干个字节，直到下一个地址能被Num整除为止。

变量如果在数据段内的偏移量是奇数，当需要读变量及其后面的字内容时，硬件将分二次读出该字内容，再拼接成一个字内容，这时，无疑需要二个读内存周期，从而影响程序执行的速度。
为了保证其偏移量是偶数，需要在其定义之前加上伪指令EVEN。

这是偶对其指令EVEN的作用。

### stosb

stosb需要寄存器edi配合使用。每执行一次stosb，就将al中的内容复制到[edi]中。 即：stosb == al --> [edi]

lodsb指令，将esi指向的地址处的数据取出来赋给AL寄存器，esi=esi+1；

lodsw指令则取得是一个字。

lodsd指令，取得是双字节，即mov eax，[esi]，esi=esi+4；

stosb指令，将AL寄存器的值取出来赋给edi所指向的地址处。mov [edi]，AL；edi=edi+1；

stosw指令取的是一个字。

stosd指令，取的是双字节，mov [edi]，eax；edi=edi+4；

### cld

cld相对应的指令是std，二者均是用来操作方向标志位DF（Direction Flag）。

cld使DF 复位，即是让DF=0，std使DF置位，即DF=1.这两个指令用于串操作指令中。通过执行cld或std指令可以控制方向标志DF，决定内存地址是增大（DF=0，向高地址增加）还是减小（DF=1，向地地址减小）。

串操作指令寻址方式有点特殊：

源操作数和目的操作数分别使用寄存器(e)si和(e)di进行间接寻址；每执行一次串操作，源指针(e)si和目的指针(e)di将自动进行修改：±1、±2、±4，其对应的分别是字节操作、字操作和双字操作。注：intel文档使用MOVSD传送双字，而GNU文档使用MOVSL传送双字。

例如：
    MOVSB //字节串传送 DF=0, SI = SI + 1 , DI = DI + 1 ；DF = 1 , SI = SI - 1 , DI = DI - 1；字串传送和双字串传送类似。
执行操作：[DI] = [SI] ,将位于DS段的由SI所指出的存储单元的字节或字传送到位于ES段的由DI 所指出的存储单元,再修改SI和DI, 从而指向下一个元素.　
    在执行该指令之前,必须预置SI和DI的初值,用STD或CLD设置DF值.
MOVS DST , SRC //同上,不常用,DST和SRC只是用来用类型检查,并不允许使用其它寻址方式来确定操作数.
1.目的串必须在附加段中,即必须是ES:[DI]
2.源串允许使用段跨越前缀来修饰,但偏移地址必须是[SI].

CLD指令功能：
将标志寄存器Flag的方向标志位DF清零。
在字串操作中使变址寄存器SI或DI的地址指针自动增加，字串处理由前往后。
例如，以下三条指令执行后，SI自动加1，更新为0001H：

```asm
CLD
MOV SI，0000H
LODSB ;将字串中的SI指针所指的一个字节装入AL
123
```

又如，以下三条指令执行后，SI自动加2，更新为0102H：

```asm
CLD
MOV SI，0100H
LODSW ;将字串中的SI指针所指的一个字(双字节)装入AX
123
```

### jnz

### cmp

cmp是比较指令， cmp的功能相当于减法指令，只是不保存结果。cmp指令执行后，将对[标志寄存器](https://baike.baidu.com/item/标志寄存器/5757541)产生影响。其他相关指令通过识别这些被影响的标志寄存器位来得知比较结果。

如果比较的是两个无符号数，则零标志位和进位标志位表示的两个操作数之间的关系如右表所示：

| CMP结果               | ZF   | CF   |
| --------------------- | ---- | ---- |
| 目的操作数 < 源操作数 | 0    | 1    |
| 目的操作数 > 源操作数 | 0    | 0    |
| 目的操作数 = 源操作数 | 1    | 0    |


如果比较的是两个有符号数，则符号标志位、零标志位和溢出标志位表示的两个操作数之间的关系如右表所示：

| CMP结果               | 标志位  |
| --------------------- | ------- |
| 目的操作数 < 源操作数 | SF ≠ OF |
| 目的操作数 > 源操作数 | SF=OF   |
| 目的操作数 = 源操作数 | ZF=1    |

CMP 指令是创建条件逻辑结构的重要工具。当在条件跳转指令中使用 CMP 时，汇编语言的执行结果就和 IF 语句一样。

这个规则真难记！

下面用三段代码来说明标志位是如何受到 CMP 影响的。设 AX=5，并与 10 进行比较，则进位标志位将置 1，原因是（5-10）需要借位：

```assembly
mov ax, 5
cmp ax,10   ; ZF = 0 and CF = 1
```

1000 与 1000 比较会将零标志位置 1，因为目标操作数减去源操作数等于 0：

```assembly
mov ax,1000
mov cx,1000
cmp cx, ax    ;ZF = 1 and CF = 0
```

105 与 0 进行比较会清除零和进位标志位，因为（105-0）的结果是一个非零的正整数。

```assembly
mov si,105
cmp si, 0    ; ZF = 0 and CF = 0
```

### zf

**CF**（carry flag）:**进位标志** 描述了最近操作是否发生了进位（可以检查无符号操作是否溢出）

**ZF**（zero flag）:**零标志** 最近操作结果为0（列如 逻辑操作 等）

**SF**（sign flag）:**符号标志**最近操作结果为负数

**OF**（overflow flag）:**溢出标志**最近操作导致一个补码溢出 补码溢出通常有两种结果（正溢出或者负溢出）

### jnz

JNZ : jump if not zero 结果不为零则转移。

JNZ（或JNE）（jump if not zero, or not equal），[汇编](https://baike.baidu.com/item/汇编)语言中的条件转移指令。结果不为零（或不相等）则转移。

格式： JNZ（或JNE） PRO

测试条件：ZF=0

下表展示了基于零标志位、进位标志位、溢出标志位、奇偶标志位和符号标志位的跳转。

| 助记符 | 说明       | 标志位/寄存器 | 助记符 | 说明       | 标志位/寄存器 |
| ------ | ---------- | ------------- | ------ | ---------- | ------------- |
| JZ     | 为零跳转   | ZF=1          | JNO    | 无溢出跳转 | OF=0          |
| JNZ    | 非零跳转   | ZF=0          | JS     | 有符号跳转 | SF=1          |
| JC     | 进位跳转   | CF=1          | JNS    | 无符号跳转 | SF=0          |
| JNC    | 无进位跳转 | CF=0          | JP     | 偶校验跳转 | PF=1          |
| JO     | 溢出跳转   | OF=1          | JNP    | 奇校验跳转 | PF=0          |

### jne

jne是一个条件转移指令。当ZF=0，转至标号处执行。

下表列出了基于相等性评估的跳转指令。有些情况下，进行比较的是两个操作数；其他情况下，则是基于 CX、ECX 或 RCX 的值进行跳转。表中符号 leftOp 和 rightOp 分别指的是 CMP 指令中的左（目的）操作数和右（源）操 作数：

| 助记符 | 说明                          |
| ------ | ----------------------------- |
| JE     | 相等跳转 (leftOp=rightOp)     |
| JNE    | 不相等跳转 (leftOp M rightOp) |
| JCXZ   | CX=0 跳转                     |
| JECXZ  | ECX=0 跳转                    |
| JRCXZ  | RCX=0 跳转（64 位模式）       |

### div

div指令是除法指令。100001/100，100001是被除数，100是除数。一般格式为：div reg或div 内存单元，reg和内存单元存放的是除数，除数可分为8位和16位2种。

被除数：默认放在AX或DX和AX，如果除数为8位，被除数则为16位，默认在AX中存放；如果除数 为16位，被除数则为32位，在DX和AX中存放，DX存放高16位，AX存放低16位。

结果：如果除数为8位，则AL存储除法操作的商，AH存储除法操作的余数；如果除数为16位，则AX存储除法操作的商，DX存储除法操作的余数。

### lea

ea是“load effective address”的缩写，简单的说，lea指令可以用来将一个内存地址直接赋给目的操作数，例如：
lea eax,[ebx+8]就是将ebx+8这个值直接赋给eax，而不是把ebx+8处的内存地址里的数据赋给eax。

而mov指令则恰恰相反，例如：
mov eax,[ebx+8]则是把内存地址为ebx+8处的数据赋给eax。

### xchg

# bochs调试程序断点问题

最近看Orange's一个操作系统的实现，其中借用了Freedos的MBR把自己写的代码植入到内存中，但是bochs断下的地方都是在FreeDos的代码里，而我想断在自己的代码中，可以进行如下设置:

```
magic_break: enabled=1
```

将上面这句话添加到bochsrc配置文件中，然后在要想中断的地方加上:

```cpp
xchg bx, bx
```



