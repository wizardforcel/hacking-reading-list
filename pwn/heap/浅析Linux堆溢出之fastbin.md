# 浅析Linux堆溢出之fastbin

> 作者：银河实验室@freebuf

> 来源：http://www.freebuf.com/news/88660.html

前端时间参加RCTF比赛，遇到了一道堆溢出的题目shaxian。漏洞本身是比较明显的，但由于对堆溢出并不熟悉，没有能够找到利用方法。之后阅读了复旦六星战队的writeup，才知道应该通过溢出来操控fastbin。由于这方面中文资料较少，于是在此将自己查阅到的相关内容进行总结整理，希望能够帮助到对此有兴趣的小伙伴。

1\. 背景知识

1.1 ptmalloc

我们都知道，在C中动态分配内存，使用的是malloc。其在GNU C(glibc)中的实现则是基于dlmalloc的ptmalloc。ptmalloc的基本思路是将堆上的内存区域划分为多个chunk，在分配/回收内存时，对chunk进行分割、回收等操作。

具体地，每个chunk除了包含最终返回用户的那部分mem，还包含头部用于保存chunk大小的相关信息。在32位系统下，chunk头的大小为8 Bytes，且每个chunk的大小也是8 Bytes的整数倍。一个典型的chunk如下图所示：

![](http://img2.tuicool.com/JFjQ32m.png!web)

#### chunk头包括以下两部分：

```
prev_size: 如果当前chunk的相邻前一chunk未被使用，prev_size为此前一chunk的大小
size: 当前chunk的大小。由于chunk大小是8的整数倍，所以此size的后3 bit被用于存储其他信息。我们需要记住的便是最低bit，即图中P的位置，用于指示前一chunk是否已被使用(PREV_INUSE)。
```

如果当前chunk处于未被使用状态，则mem前8 bytes被用来存储其他信息，具体如下：

```
fd: 下一个未被使用的chunk的地址
bk: 上一个未被使用的chunk的地址
```

可以看到，chunk头中包含的大小信息，主要用来在获取内存中相邻chunk的地址（当前chunk地址减去前一chunk的大小，为前一chunk的地址；当前chunk地址加上当前chunk的大小，为后一chunk的地址）。而mem中的fd和bk只在当前chunk处于未被使用时才有意义。如果了解数据结构，便可以立刻看出，这些未被使用的chunks通过fd, bk组成了链表。事实上，malloc确实维护了一系列链表用于内存的分配和回收，这些链表被成为"bins"。

一般来说，每个bin链表中的chunk都有相同或将近的大小。根据bin所包含chunk的大小，可以将bin分为fastbin, unsorted bin, small bin, large bin。我们这里要研究的就是fastbin。

1.2 fastbin

fastbin所包含chunk的大小为16 Bytes, 24 Bytes, 32 Bytes, … , 80 Bytes。当分配一块较小的内存(mem&lt;=64 Bytes)时，会首先检查对应大小的fastbin中是否包含未被使用的chunk，如果存在则直接将其从fastbin中移除并返回；否则通过其他方式（剪切top chunk）得到一块符合大小要求的chunk并返回。

而当free一块chunk时，也会首先检查其大小是否落在fastbin的范围中。如果是，则将其插入对应的bin中。顾名思义，fastbin为了快速分配回收这些较小size的chunk，并没对之前提到的bk进行操作，即仅仅通过fd组成了单链表而非双向链表，而且其遵循后进先出(LIFO)的原则。

举例来说，假设目前大小为40 Bytes的fastbin中已经包含了一个位于0x0804a000的chunk。

![](http://img0.tuicool.com/VnQj2ib.png!web)

当另一块大小为40 Bytes，位于0x0804a028的chunk被free时，其被放至同一fastbin中。具体地，0x0804a028成为该fastbin的首个chunk，之前的首个chunk 0x0804a000，则被保存于0x0804a028的fd中。

![](http://img1.tuicool.com/QjMfMn.png!web)

接下来，调用malloc分配一块32 Bytes的内存(实际大小为40 Bytes的chunk)时，该fastbin中的首个chunk, 0x0804a028会被移除并返回。此时该fastbin的首个chunk变为0x0804a028的fd内容，即0x0804a000。此时便恢复到之前的状态。

当然，在实际执行分配或回收时，还会对目标chunk的大小进行检查。但如果能够修改fd内容，那么在随后的malloc时便可能将修改后的地址返回，这进一步往往能够造成向任意地址写任意内容(write-anything-anywhere)的后果。

2\. 实例

我们编写一个存在问题的简单程序如下：

```
#include#include#include
int size = 40 | 0x1;

int main(int argc, char *argv[]) {
    void *buf0, *buf1, *buf2;
    buf0 = malloc(32);
    buf1 = malloc(32);

    free(buf1);
    free(buf0);

    buf0 = malloc(32);
    read(0, buf0, 64);
    buf1 = malloc(32);

    buf2 = malloc(32);

    printf("buf2 is at %p\n", buf2);

    return 0;
}
```

可以看到，该代码在调用read时，向buf0写入的内容超过了其本身的大小，发生了堆溢出。我们可以利用此处漏洞，修改相邻chunk的fd内容，造成在为buf2分配内存时，从fastbin返回得到非正常的地址。

具体地，在调用read之前，fastbin的结构如下图所示：

![](http://img0.tuicool.com/NvuUrqz.png!web)

而如果我们利用read时的溢出，修改其后方buf1中fd的内容，那么在之后buf1=malloc(32)时，0x0804a028会被从fastbin中取出并返回。而且篡改的fd会被放入该fastbin，其指向的伪chunk会在之后buf2=malloc(32)时返回。值得注意的是，由于在分配时还会检查从fastbin中取出的chunk大小是否符合要求，因此我们的伪chunk的size也为0×29。恰好存在一个全部变量的值为0×29，我们便取其地址减4处，即0x080497e8，作为伪chunk的地址：

![](http://img1.tuicool.com/ZneQZvv.png!web)

可以通过以下命令查看我们溢出的效果：

![](http://img1.tuicool.com/quQbAbi.png!web)

输出的结果显示，buf2会被分配至0x80497f0，而这里恰为伪chunk所对应的mem。

一般地，我们往往可以向分配得到的内存中写入数据。因此如果malloc返回的地址可被控制，那么便可实现write-anything-anywhere的效果。

3\. House of Spirit

事实上，linux下堆溢出的攻击利用，早在十年前便已有人已深入研究。2005年，一篇名为"The Malloc Maleficarum"的文章便提出了5种攻击堆的方式：

```
The House of Prime
The House of Mind
The House of Force
The House of Lore
The House of Spirit
The House of Chaos
```

随后在2009年，Phrack 66期上也刊登了一篇名为"Malloc Des-Maleficarum"的文章，对这几种技术进行了进一步的分析。

在这其中，House of Spirit是与fastbin相关。因此在这里，我们也对这种攻击方式结合自己的理解进行简要介绍。

House of Spirit实现的最终效果，也是使攻击者构造的伪chunk通过fastbin被malloc返回。与之前例子中的溢出覆盖fd不同的是，House of Spirit是通过篡改free的目标地址，将伪chunk放入fastbin，进而使随后的malloc返回此伪chunk。

#### 举例来说，如果存在以下代码片段：

```
#include#include#include
int main(int argc, char *argv[]) {
    void *p = malloc(32);
    char buf[8];
    read(0, buf, 0x80);

    free(p);
    malloc(32);
}
```

可以看到，在read数据至buf时，可以溢出修改指针p的内容。如果知道了栈的地址，那么便可在read时构造伪chunk，并将p修改为伪chunk对应的mem地址。在测试用例中，buf位于0xffffd704，我们的伪chunk大小为40 Bytes，位于0xffffd728（注意chunk地址按8对齐），对应的mem则位于0xffffd730。

此外，glibc中在free时，还会对相邻后一个chunk的大小进行检查，如下图中行3901、3902所示：

![](http://img1.tuicool.com/22uuI3Q.png!web)

因此我们在伪chunk之后，需要另一个伪chunk。这里只需其大小不过大或过小即可。最简单的方式如下图所示：

![](http://img1.tuicool.com/yqEj2eI.png!web)

将payload1作为输入使用gdb调试，发现在free(p)之后，fastbin的内容如下：

![](http://img0.tuicool.com/RrIfYfr.png!web)

可以看到，我们位于0xffffd728的伪chunk现在确实被放置于对应的fastbin中。在随后的malloc(32)，返回得到的地址便为栈上的伪mem, 0xffffd730。

4\. 扩展

在研究完相关技术后，我便立刻搜索这种技术曾经在现实中的exploit被使用。可惜并没有找到直接对应的exploit（可能我的搜索方式不准确，如果有发现的还请告诉我）。

但是，如果我们将问题再抽象一下，便可发现这些技术利用的本质即为：allocator所需的chunk信息被溢出等方式修改，造成随后分配得到的地址为构造的非正常地址。如果以这种方式来回顾，那我们首先想到的便是不久前闹得沸沸扬扬的GHOST(CVE-2015-0235)。这一漏洞的深入分析可见https://www.qualys.com/2015/01/27/cve-2015-0235/GHOST-CVE-2015-0235.txt，在这里我们便不赘述了，仅对与内存分配相关的利用方式进行简要回顾。

GHOST漏洞利用的其中一环，便是"write-anything-anywhere"。而这是由Exim的内存分配引起的。具体地，Exim自己实现的内存分配器，使用了如下结构：

```
typedef struct storeblock {
  struct storeblock *next;
  size_t length;
} storeblock;
```

通过之前的溢出，覆盖这一结构体，进而修改next指针。其后果便是，此篡改的地址会被作为下一块可用的内存，被之后的内存分配返回。在随后的写入时便造成了"write-anything-anywhere"。

5\. 总结

堆溢出的利用方式有很多，除了这里介绍的fastbin，还有大名鼎鼎的unlink。虽然利用的细节不同，但大多是围绕着对分配器内部数据结构的篡改展开。这篇文章仅仅是抛砖引玉，希望大牛们多多分享，共同提高。

6\. 参考

[https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/](https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/)

[https://gbmaster.wordpress.com/2014/08/24/x86-exploitation-101-this-is-the-first-witchy-house/](https://gbmaster.wordpress.com/2014/08/24/x86-exploitation-101-this-is-the-first-witchy-house/)

[http://packetstorm.foofus.com/papers/attack/MallocMaleficarum.txt](http://packetstorm.foofus.com/papers/attack/MallocMaleficarum.txt)

[http://phrack.org/issues/66/10.html](http://phrack.org/issues/66/10.html)

[https://www.qualys.com/2015/01/27/cve-2015-0235/GHOST-CVE-2015-0235.txt](https://www.qualys.com/2015/01/27/cve-2015-0235/GHOST-CVE-2015-0235.txt)