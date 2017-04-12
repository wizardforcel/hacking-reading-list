# Nmap Cheat Sheet Part 1

> 译者：未知

> 原文：[Nmap Cheat Sheet: From Discovery to Exploits – Part 1: Introduction to Nmap](http://resources.infosecinstitute.com/nmap-cheat-sheet/)

在侦查期间，扫描一直是信息收集的初始阶段。

## **什么是侦查**

侦查是尽可能多收集关于目标网络的信息。从黑客的角度来看，信息收集对于一次攻击非常有用，所以为了封锁恶意的企图，渗透测试者通常尽力查找这些信息，发现后修复这个缺陷。这也被叫做踩点。通过信息收集，人们通常会发现如下类型的信息：

+  E-mail 地址

+ 端口号/协议

+ 操作系统信息

+ 运行的服务

+ Traceroute 信息/DNS 信息

+ 防火墙标识和绕过

+ 其他

所以对于信息收集，扫面是第一个部分。在扫面阶段，Nmap对于发现开放端口，协议号，操作系统信息，防火墙信息等是一个非常有用的工具。

## **Nmap 介绍**

Nmap (网络映射器)是一个开源工具，它使网络探测和安全审计得以专业化。最初由 Gordon “Fyodor” Lyon 发布。官网官方网站是[http://nmap.org](http://nmap.org). Nmap是一个免费的用来实现网络探测和安全审计的开源程序。许多系统和网络管理员发现它对于一些日常的工作也有帮助。例如查看整个网络的信息，管理服务升级计划以及监控主机和服务的正常运行。

Nmap采用一种新颖的方式利用原始IP包来决定网络上是什么样的主机，这些主机提供什么样的服务（应用程序名和版本），它们运行着什么样的操作系统（操作系统版本）它们使用什么类型的过滤器/防火墙以及许多其他的特征。它虽然被设计用来快速扫描大型网络，但是在单个主机上也会工作的非常好。Nmap可以运行在所有的主流计算机操作系统上，Linux，Windows，Mac OS X都可以找到官方的安装包。

## **安装Nmap**

Nmap 对于不同的环境有着非常好的支持。

Windows: 从官方网站安装 [http://nmap.org](http://nmap.org)。windows上的所有图形用户界面和命令行都可以从官网上找到。Nmap的图形用户界面采用的是Zenmap。

Linux (Ubuntu and Debian): 在Linux终端上运行如下命令：`apt-get install nmap`

如下的图片中，我已经安装好了Nmap。

[![](http://image.3001.net/images/20140417/13977455645414.png!small "1.png")](http://image.3001.net/images/20140417/13977455645414.png)

![](http://image.3001.net/images/20140417/13977455645414.png!small)

基于Red Hat 和 Fedora 的系统: `yum install nmap`

基于Gentoo Linux 的系统： `emerge nmap`

接下来，我所有的操作将会用Linux终端来演示：

## **Nmap脚本引擎**

Nmap脚本引擎 (NSE) 是Nmap最有力灵活的的一个特性。它允许用户撰写和分享一些简单的脚本来一些较大的网络进行扫描任务。基本上这些脚本是用Lua编程语言来完成的。通常Nmap的脚本引擎可以完成很多事情，下面是其中的一部分：

### **网络探测**

这是Nmap的基础功能。例如查询目标域名的WhoIs数据，查询目标IP的ARIN, RIPE, 或者 APNIC 来确定所有者，对开放端口执行鉴别查询，SNMP 查询以及列出可用的NFS/SMB/RPC 分享和服务。

### **漏洞检测**

当一个新的漏洞被发现，你会想要在坏家伙行动之前快速扫描你的网络来识别含有漏洞的系统。因为Nmap不是一个专业的漏洞扫描器，而NSE用来处理这种需求的漏洞检查是足够的。现在已经有很多漏洞检测脚本可用，并且他们计划发布更多。

许多攻击者和一些自动化的蠕虫留下一些后门使得攻击者之后能重入。他们中的等一些可用被Nmap的正则表达式版本检测到。

### **漏洞利用**

作为一个通用的脚本语言，可以使用NSE来执行漏洞利用而不仅仅局限于发现漏洞。这种添加自定义漏洞利用脚本的功能对于一些人（尤指渗透测试者）是很有价值的，尽管他们不打算将Nmap变成一个像Metasploit一样的漏洞利用框架。

正如下面你将会看到的，我已经使用了（-sc）选项（或者-script），它是一个扫描目标网络的默认脚本。你可以看到我们得到了ssh，rpcbind，netbios-sn，但是端口是过滤的或者关闭的，所以我们可以说这里可能有一个防火墙来封堵我们的请求。稍后我们将会讨论怎么识别防火墙并尝试绕过它们。

[![](http://image.3001.net/images/20140417/13977455652962.png!small "2.png")](http://image.3001.net/images/20140417/13977455652962.png)

![](http://image.3001.net/images/20140417/13977455652962.png!small)

现在我已经用探测模式运行了一个ping扫描脚本，它将会尝试所有可能扫描的方法，这样我们将会得到更多的有用的信息。

[![](http://image.3001.net/images/20140417/13977455651015.png!small "3.png")](http://image.3001.net/images/20140417/13977455651015.png)

![](http://image.3001.net/images/20140417/13977455651015.png!small)

正如你在图片中看到的，它将会尝试按照脚本规则的所有可能的办法。在下一张途中可以看到更多信息。

你能看到有趣的协议和端口么？你可以看到dns-bruteforce发现了一个包含一些博客，内容管理系统，sql，记录，邮件以及许多其他信息的主机。所以我们可以执行sql注入，这个博客可能是WordPress，Joomal等等。我们可以利用一些已知的CMS漏洞执行攻击，很明显这些方法将会是一个黑盒测试。

在下面的一节中，我将会阐述怎么撰写你自己的Nmap脚本引擎，以及怎样用Nmap来利用他们。

## **基本的扫描技术**

接下来我将会演示一些基本的扫描技术。但是在那之前，你应该知道一些关于Nmap扫描后状态的的基本知识。

端口状态：扫描之后，你可能会看到一些端口状态如 open(开放的)，filtered(被过滤的)，closed(关闭的)等等。我来对此解释下。

+ Open(开放的): 应用程序正在这个端口上监听连接。

+ Closed(关闭的): 端口对探测做出了响应，但是现在没有应用程序在监听这个端口。

+ Filtered(过滤的): 端口没有对探测做出响应。同时告诉我们探针可能被一些过滤器（防火墙）终止了。

+ Unfiltered(未被过滤的):端口对探测做出了响应，但是Nmap无法确定它们是关闭还是开放。

+ Open/Filtered: 端口被过滤或者是开放的，Nmap无法做出判断。

+ Closed/Filtered: 端口被过滤或者是关闭的，Nmap无法做出判断。

## **开始扫描主机**

### **扫描单个网络**

在你的Nmap(Windows/Linux)上运行如下命令: `nmap 192.168.1.1(or) host name`。

[![](http://image.3001.net/images/20140417/13977455685907.png!small "5.png")](http://image.3001.net/images/20140417/13977455685907.png)

![](http://image.3001.net/images/20140417/13977455685907.png!small)

### **扫描多个网络目标**

你可以用Nmap扫描多个目标来发现主机或者收集信息。

命令: `nmap host1 host2 host3 etc….` 它会扫面不同的IP地址和整个子网。

[![](http://image.3001.net/images/20140417/13977455715195.png!small "6.png")](http://image.3001.net/images/20140417/13977455715195.png)

![](http://image.3001.net/images/20140417/13977455715195.png!small)

你也可以用相同的命令一次扫描多个网络网站域名。如下图所示。它将会把域名转换成对应的IP地址，并扫描它。

[![](http://image.3001.net/images/20140417/13977455724413.png!small "7.png")](http://image.3001.net/images/20140417/13977455724413.png)

![](http://image.3001.net/images/20140417/13977455724413.png!small)

### 扫描IP地址段

命令:`nmap 192.168.2.1-192.168.2.100`

Nmap 可以用来扫描用CIDR格式表示的整个子网。

### **使用语法: nmap [CIDR格式的网络地址]**

例如:`nmap 192.168.2.1/24`

### **扫描目标列表**

如果你有大量的系统需要扫描，你可以将这些IP地址（或主机名）输入到一个文本文件中，用这个文本文件的内容来作为Nmap命令行的输入。

语法: `nmap -iL [list.txt]`

### **扫描随机目标**

“-iR” 参数可以用来选择随机的互联网主机来扫描。Nmap将会随机的生成指定数量的目标进行扫描。

语法: `nmap -iR [主机数量]`

除非你有一些这种类型的任务，否则随机扫描不是一个好习惯。

### **" –exclude "选项被用来从扫描中排除一些主机**

语法: `nmap [目标] –exclude [主机]`

例如:`nmap 192.168.2.1/24 –exclude 192.168.2.10`

### **激烈扫描**

激烈扫描模式会选择Nmap中最常用的选项来尝试代替输入很长的字符串。它对于路由跟踪也适用。

命令:`nmap –A host`

### **用Nmap来探测**

用Nmap来探测对于渗透测试者来说是非常有意思而且非常有用的。在探测中，可以发现服务，端口号，防火墙，协议，操作系统等等。我们解析来一个一个的讨论。

### **不用ping**

" -PN "选项告诉Nmap不使用默认的探测检查，而是对目标进行一个完整的端口扫描。当我们扫描一个有防火墙保护而封锁 ping 探针主机的时候是非常有用的。

语法:`nmap –PN Target`

[![](http://image.3001.net/images/20140417/13977455737602.png!small "8.png")](http://image.3001.net/images/20140417/13977455737602.png)

![](http://image.3001.net/images/20140417/13977455737602.png!small)

通过指定这些选项，Nmap将会不用ping来发现那些不能ping通主机开放的端口。

### **只是扫描的ping**

“ -Sp ” 选项告诉Nmap仅仅进行ping扫描。 当你有一组IP地址来扫描时，而且你不知道哪一个是可达的，这时这个选项会很有用。通过指定一个特定的目标，你可以得到像MAC地址这样更多的信息。

语法:`nmap –Sp target`

[![](http://image.3001.net/images/20140417/13977455736608.png!small "9.png")](http://image.3001.net/images/20140417/13977455736608.png)

![](http://image.3001.net/images/20140417/13977455736608.png!small)

### **TCP SYN 扫描**

在我们开始之前，我们必须知道syn包。

基本上来说，一个syn包是用来在两个通信的主机之间初始化连接。

TCP SYN ping 发送一个SYN包给目标系统，然后监听目标系统的响应。 这种探测方法对于那些配置好封锁标准ICMP ping的系统来说很有用。

“ -PS ”选项来实施 TCP SYN ping。

语法:`nmap –PS 目标`

[![](http://image.3001.net/images/20140417/13977455758582.png!small "10.png")](http://image.3001.net/images/20140417/13977455758582.png)

![](http://image.3001.net/images/20140417/13977455758582.png!small)

默认的端口是80端口，你也可以指定其他的端口，例如：–PS22, 23, 25, 443。

### **TCP Ack Ping 扫描**

这种类型的扫描将只会扫描ACK包。

“ -PA ”在特定目标上进行一个 TCP ACK ping。

“ -PA ”选项会导致Nmap发送一个 TCP ACK 包给指定的主机。

语法:`nmap –PA target`

[![](http://image.3001.net/images/20140417/13977455785137.png!small "11.png")](http://image.3001.net/images/20140417/13977455785137.png)

![](http://image.3001.net/images/20140417/13977455785137.png!small)

这种方法将会通过对一个TCP连接作出响应来尝试发现主机，这个TCP连接是一个不存在的连接，它正试图与目标主机建立一个响应。如同其他ping 选项一样，对于封锁标准ICMP ping的情况是非常有用的。

### **UDP Ping 扫描**

“ –PU ”扫描只会对目标进行 udp ping 扫描。这种类型的扫描会发送UDP包来获得一个响应。

语法:`nmap –PU target`

[![](http://image.3001.net/images/20140417/1397745581801.png!small "12.png")](http://image.3001.net/images/20140417/1397745581801.png)

![](http://image.3001.net/images/20140417/1397745581801.png!small)

你可以指定一个扫描的端口号，例如 –PU 22, 80, 25, 等等。在上面的图片中，目标是我的局域网IP，在这个IP的主机中没有任何UDP服务。

### **Sctp 初始 ping**

“ -PY ”参数告诉Nmap进行一个 SCTP 初始 ping. 这个选项将会发送一个包含最小的初始快STCP包。这种探测方法会尝试用SCTP来定位主机。SCTP通常被用在IP拨号服务的系统中。

语法:`nmap –PY 目标`

[![](http://image.3001.net/images/20140417/13977455835534.png!small "13.png")](http://image.3001.net/images/20140417/13977455835534.png)

![](http://image.3001.net/images/20140417/13977455835534.png!small)

在上图中，尽管在这台主机上没有sctp服务，但是我们必须用-pn选项来进行探测。

### **ICMP 回声应答 ping**

“ -PE ”参数进行一个ICMP(Internet控制报文协议) 在指定的系统上输出ping。

语法:`nmap –PE 目标`

[![](http://image.3001.net/images/20140417/13977455842018.png!small "14.png")](http://image.3001.net/images/20140417/13977455842018.png)

![](http://image.3001.net/images/20140417/13977455842018.png!small)

这种类型的探测在ICMP数据包可以在有较少传输限制的系统上效果比较好。

### **ICMP时间戳ping扫描**

“ -PP ”选项进行一个ICMP时间戳ping扫描。

[![](http://image.3001.net/images/20140417/13977455878203.png!small "15.png")](http://image.3001.net/images/20140417/13977455878203.png)

![](http://image.3001.net/images/20140417/13977455878203.png!small)

### **ICMP地址掩码ping**

“ -PM ”选项进行一个ICMP地址掩码ping扫描。

语法:`nmap –PM 目标`

[![](http://image.3001.net/images/20140417/13977455879506.png!small "16.png")](http://image.3001.net/images/20140417/13977455879506.png)

![](http://image.3001.net/images/20140417/13977455879506.png!small)

这种非常规的ICMP查询（和 -PP 选项类似）试图用备选的ICMP登记ping指定的主机。这种类型的ping可以偷偷的通过配置好封锁标准回声请求的防火墙。

### **IP协议Ping**

“ -PO ”选项进行一个IP协议ping。

语法:`nmap –PO 协议 目标`

[![](http://image.3001.net/images/20140417/13977455915677.png!small "17.png")](http://image.3001.net/images/20140417/13977455915677.png)

![](http://image.3001.net/images/20140417/13977455915677.png!small)

IP协议ping用指定的协议发送一个包给目标。如果没有指定协议，默认的协议是 1 (ICMP), 2 (IGMP)和4 (IP-in-IP)。

### **ARP ping**

“ –PR ”选项被用来实施一个arp ping 扫描。“ -PR ”选项告诉Nmap对目标主机进行一个APR(地址解析协议) ping.

语法: `nmap –PR 目标`

[![](http://image.3001.net/images/20140417/13977455986458.png!small "18.png")](http://image.3001.net/images/20140417/13977455986458.png)

![](http://image.3001.net/images/20140417/13977455986458.png!small)

“ -PR ”选项当扫描整个网络的时候自动使用。这种类型的探测比其他的ping方法更快。

路由跟踪

“ –traceroute ”参数可以用来追踪到指定主机的网络路径。

语法: `nmap –traceroute 目标`

[![](http://image.3001.net/images/20140417/13977455992946.png!small "19.png")](http://image.3001.net/images/20140417/13977455992946.png)

![](http://image.3001.net/images/20140417/13977455992946.png!small)

### **强制使用反向域名解析**

“ -R ”参数告诉Nmap总是对目标IP地址实施一个逆向DNS解析。

语法: `nmap –R 目标`

[![](http://image.3001.net/images/20140417/13977456005474.png!small "20.png")](http://image.3001.net/images/20140417/13977456005474.png)

![](http://image.3001.net/images/20140417/13977456005474.png!small)

“ -R “选项可以用在对一个IP地址块实施探测的时候，Nmap将试图对每个IP地址进行反向向DNS信息解析。

### **不用反向域名解析**

“ -n ”参数用来说明不使用反向域名解析。

语法:`nmap –n 目标`

[![](http://image.3001.net/images/20140417/13977456006250.png!small "21.png")](http://image.3001.net/images/20140417/13977456006250.png)

![](http://image.3001.net/images/20140417/13977456006250.png!small)

反向域名解析会明显的降低Nmap扫描的速度。使用“-n”选项会极大的减少扫描时，特别是当扫描大量的主机时。如果你不关心目标系统的DNS信息，更喜欢进行一个能快速产生结果的扫描时，可以使用这个选项。

### **DNS查询方法的取舍**

“ –system-dns ”选项告诉Nmap使用主机系统的域名解析来替代它自己的内部方法。

语法:`nmap –system-dns 目标`

[![](http://image.3001.net/images/20140417/13977456015443.png!small "22.png")](http://image.3001.net/images/20140417/13977456015443.png)

![](http://image.3001.net/images/20140417/13977456015443.png!small)

### **手动指定DNS服务器**

“ –dns-servers ”选项可以用来在扫面的时候手动指定查询的DNS服务器。

语法: `nmap –dns-servers 服务器1 服务器2 目标`

**[![](http://image.3001.net/images/20140417/13977456023808.png!small "23.png")](http://image.3001.net/images/20140417/13977456023808.png)**

![](http://image.3001.net/images/20140417/13977456023808.png!small)</strong>

” –dns-servers “选项允许你指定一个或者更多的替代服务器来宫Nmap查询。这对于没有DNS配置的系统是非常有用的，而且如果你想阻止你的扫描查询出现在你配置在本地DNS服务器的记录文件中，这个选项也是有用的。

### **列表扫描**

” -sL “选项将会显示一个列表，并对指定的IP地址执行一个反向DNS查询。

语法:`nmap –sL 目标`

[![](http://image.3001.net/images/20140417/13977456032105.png!small "24.png")](http://image.3001.net/images/20140417/13977456032105.png)

![](http://image.3001.net/images/20140417/13977456032105.png!small)

在下一个I部分中，我将会讨论怎么用不同的方法发现服务，主机和旗标。同时，我们也将会讨论怎么发现防火墙以及怎么通过使用Nmap的NSE来绕过它以及怎么编写你自己的Nmap脚本引擎。Nmap中最重要的部分是知道怎么样发现漏洞和利用他们，拭目以待。

## **参考**

[http://nmap.org/](http://nmap.org/)

via [infosecinstitute](http://resources.infosecinstitute.com/nmap-cheat-sheet/)