# CTF中图片隐藏文件分离方法总结

> 来源：https://www.hackfun.org/learnrecords/summary-of-image-hiding-files-in-CTF.html

## 0x00 前言

在安全的大趋势下，信息安全越来越来受到国家和企业的重视，所以CTF比赛场次越来越多，而且比赛形式也不断的创新，题目也更加新颖有趣，对选手的综合信息安全能力有一个较好的考验，当然更好的是能从比赛有所收获，不断学习和总结提升自己的信息安全能力与技术。转到CTF比赛上，通常在CTF比赛中常有与隐写术(Steganography)相关的题目出现，这里我们讨论总结图片隐藏文件分离的方法，欢迎大家补充和交流:P

## 0x01 分析

这里我们以图片为载体，给了这样的一样图片：

![](http://img0.tuicool.com/QVVreiR.jpg!web)

首先我们需要对图片进行分析，这里我们需要用到一个工具 [binwalk](https://github.com/devttys0/binwalk) ，想要了解这个工具可以参考这篇 [Binwalk：后门（固件）分析利器](http://www.freebuf.com/sectool/15266.html) 文章，以及 [kali官方对binwalk的概述和使用介绍](http://tools.kali.org/forensics/binwalk) 。

这里我们就是最简单的利用，在binwalk后直接提供固件文件路径和文件名即可:

```
# binwalk carter.jpg
```

当我们使用这行命令后，binwalk就会自动分析这个jpg文件：

```
# binwalk carter.jpg

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JPEG image data, JFIF standard 1.01
382           0x17E           Copyright string: "Copyright (c) 1998 Hewlett-Packard Company"
3192          0xC78           TIFF image data, big-endian, offset of first image directory: 8
140147        0x22373         JPEG image data, JFIF standard 1.01
140177        0x22391         TIFF image data, big-endian, offset of first image directory: 8
```

从上面的内容显然看得出来这个jpg文件还隐藏着另一个jpg文件，从140147块偏移开始就是另一张jpg。

## 0x02 分离

在得到隐藏信息之后我们下一步就是把另一张jpg分离出，以下讨论几种方法：

#### （1）使用dd命令分离(linux/unix下)

我们可以使用dd命令分离出隐藏文件：

```
# dd if=carter.jpg of=carter-1.jpg skip=140147 bs=1
```

可以参考 [dd命令详解](http://www.cnblogs.com/qq78292959/archive/2012/02/23/2364760.html) ，这里if是指定输入文件，of是指定输出文件，skip是指定从输入文件开头跳过140147个块后再开始复制，bs设置每次读写块的大小为1字节 。

最后我们可以得到这样的一张carter-1.jpg图片：

![](http://img0.tuicool.com/maqeiyb.jpg!web)

#### （2）使用foremost工具分离

foremost是一个基于文件文件头和尾部信息以及文件的内建数据结构恢复文件的命令行工具，win可以下载地址，Linux可以

通过下面命令安装使用：

```
# apt-get install foremost
```

安装foremost后你可以使用foremost -help查看使用帮助，这里最简单分离文件的命令为：

```
# foremost carter.jpg
```

当我们使用这行命令后，foremost会自动生成output目录存放分离出文件：

![](http://img2.tuicool.com/73Ub2iF.png!web)

#### （3）hex编辑器分析文件

至于hex编辑器有很多，win下有用得较多的winhex,UltraEdit等，linux下有hexeditor等，这里我们以winhex为例手动分离，在分离之前我们需要知道一点关于jpg文件格式的知识，jpg格式文件开始的2字节是图像开始SOI(Start of Image,SOI)为FF D8，之后2个字节是JFIF应用数据块APPO(JFIF application segment)为FF E0 ，最后2个字节是图像文件结束标记EOI(end-of-file)为FF D9 ，如果你想详细了解更多关于这方面的知识可以参考jpg文件格式分析一文。

用winhex打开图片，通过Alt+G快捷键输入偏移地址22373跳转到另一张jpg的图像开始块，可以看到FF D8图像开始块。

![](http://img2.tuicool.com/QFjmuqr.png!web)

而图像结束块FF D9

![](http://img0.tuicool.com/aqmqe2z.png!web)

选取使用Alt+1快捷键选取FF为开始的块，Alt+2选取D9为结束块，然后右键-&gt;Edit-&gt;Copy Block-&gt;Into New File保存相应的文件后缀，例如new.jpg

![](http://img1.tuicool.com/IvYJVbB.png!web)

## 0x03 其他

还有一种特例，它是事先制作一个hide.zip，里面放入隐藏的文件，再需要一张jpg图片example.jpg，然后再通过命令 copy /b example.jpg+hide.zip output.jpg生成output.jpg的新文件，原理是利用了copy命令，将两个文件以二进制方式连接起来，正常的jpg文件结束标志是FF D9，而图片查看器会忽视jpg结束符之后的内容，所以我们附加的hide.zip就不会影响到图像的正常显示。(参考AppLeU0的 [隐形术总结](http://drops.wooyun.org/tips/4862) )

针对这种特例我们可以直接将jpg文件改为zip文件后缀(其他文件如rar文件也类似)，就可以看到hide.zip压缩包里隐藏的文件。

比如当我们得到一张wh3r3_is_f14g.jpg文件：

![](http://img2.tuicool.com/byqU7nY.jpg!web)

当我们用winhex打开文件，发现wh3r3_is_f14g.jpg文件最后数据块不是FF D9 jpg文件的结束标志，而是zip文件的结束标志。

![](http://img2.tuicool.com/IBbuY3N.png!web)

我们直接将文件改名为wh3r3_is_f14g.zip，打开得到flag.txt。

![](http://img2.tuicool.com/Vj2aQrE.png!web)

最后打开flag.txt得到flag。

![](http://img0.tuicool.com/ZjIJrqZ.jpg!web)

## 0x03 后话

图片隐写方式有很多种，在此只介绍了这一种，如果以后有机会会写其他的图片隐写，如果对隐写感兴趣这里推荐一本机械工业出版社的《数据隐藏技术揭秘：破解多媒体、操作系统、移动设备和网络协议中的隐秘数据》，如果你不想购买实体书，可以 [下载pdf版](http://www.jb51.net/books/434273.html) 。

这里我把所有图片打包了zip，如果有需要自行下载吧: P

[Steganography_Pictures.zip](https://www.hackfun.org/usr/uploads/2016/07/3701056190.zip)