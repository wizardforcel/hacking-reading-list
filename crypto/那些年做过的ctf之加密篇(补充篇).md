# 那些年做过的ctf之加密篇(补充篇)

> 作者：4ido10n

> 来源：http://www.secbox.cn/hacker/ctf/8078.html

在乌云上看到**[好基友一辈子](http://drops.wooyun.org/author/%E5%A5%BD%E5%9F%BA%E5%8F%8B%E4%B8%80%E8%BE%88%E5%AD%90)**写了[那些年做过的ctf之加密篇](http://drops.wooyun.org/tips/10002)，感觉挺不错的，首先感谢原作者分享，之前也想过写一篇类似的文章，但是迟迟没有动手，正好看到好基友一辈子的文章，有些可能作者没有提到，我就补充一些，也欢迎大家补充: p

**0x01 Base64**

Base64:

```
ZXZhbCgkX1BPU1RbcDRuOV96MV96aDNuOV9qMXVfU2gxX0oxM10pNTU2NJC3ODHHYWJIZ3P4ZWY=
```

Base64编码要求把3个8位字节（3*8=24）转化为4个6位的字节（4*6=24），之后在6位的前面补两个0，形成8位一个字节的形式。 如果剩下的字符不足3个字节，则用0填充，输出字符使用'='，因此编码后输出的文本末尾可能会出现1或2个'='

Base32: Base32和Base64相比只有一个区别就是，用32个字符表示256个ASC字符，也就是说5个ASC字符一组可以生成8个Base字符，反之亦然。

base64[在线编解码](http://base64.xpcha.com/)

当然还有base32和base16加密，base64全家桶可以用python里的base64模块来搞定。

[参考链接1](https://docs.python.org/3.3/library/base64.html)

[参考链接2](http://www.cnblogs.com/xiaowuyi/archive/2012/05/31/2528608.html)

**0x02 希尔密码**

> 希尔密码：密文： 22,09,00,12,03,01,10,03,04,08,01,17 （明文：wjamdbkdeibr）

解题思路：使用的矩阵是 1 2 3 4 5 6 7 8 10

更多请参考[原文链接](http://bobao.360.cn/ctf/learning/136.html)

详见[百度百科](http://baike.baidu.com/link?url=R6oWhCdKvzlG8hB4hdIdUT1cZPbFOCrpU6lJAkTtdiKodD7eRTbASpd_YVfi4LMl7N8yFyhVNOz5ki6TC7_5eq)

**0x03 栅栏密码**

> 栅栏密码：把要加密的明文分成N个一组，然后把每组的第1个字连起来，形成一段无规律的话。

密文样例：`tn c0afsiwal kes,hwit1r  g,npt  ttessfu}ua u  hmqik e {m,  n huiouosarwCniibecesnren.`

解密程序：

```
char s[]= "tn c0afsiwal kes,hwit1r  g,npt  ttessfu}ua u  hmqik e {m,  n huiouosarwCniibecesnren.";  
char t[86]= "";  
int i,j,k;
k=0;
for (i=0;i&amp;lt;17;i++)  
{  
      for(j=0;j&amp;lt;5;j++)  
      {  
                t[k++]= ch[j*17+i];  
      }  
}  
for(i=0;i&amp;lt;85;i++)
{
    printf("%c",t[i]);
}  
```

更多请参考[原文链接](http://blog.csdn.net/shinukami/article/details/45980629) [百度百科](http://baike.baidu.com/link?url=WTLCkG-FGup9zjcMTjeRkyXVOFIjcaBCCojrmdkWrxym-H2mWPQH06LsbfSSmgESC_AoEJ6G1uun_5VT5MpTMa)

**0x04 凯撒密码**

[![那些年做过的ctf之加密篇(补充篇)-安全盒子](http://www.secbox.cn/wp-content/uploads/2015/11/凯撒密码加密.jpg)](http://www.secbox.cn/wp-content/uploads/2015/11/凯撒密码加密.jpg "那些年做过的ctf之加密篇(补充篇)")

> 凯撒密码：通过把字母移动一定的位数来实现加密和解密。明文中的所有字母都在字母表上向后（或向前）按照一个固定数目进行偏移后被替换成密文。

密文样例：

```
U8Y]:8KdJHTXRI>XU#?!K_ecJH]kJG*bRH7YJH7YSH]*=93dVZ3^S8*$:8"&:9U]RH;g=8Y!U92'=j*$KH]ZSj&[S#!gU#*dK9\.
```

解题思路：得知是凯撒加密之后，尝试进行127次轮转爆破：

```
lstr="""U8Y]:8KdJHTXRI>XU#?!K_ecJH]kJG*bRH7YJH7YSH]*=93dVZ3^S8*$:8"&:9U]RH;g=8Y!U92'=j*$KH]ZSj&[S#!gU#*dK9\."""</p>
for p in range(127):
str1 = ''
for i in lstr:
temp = chr((ord(i)+p)%127)
if 32<ord(temp)<127 :
str1 = str1 + temp
feel = 1
else:
feel = 0
break
if feel == 1:
print(str1)
```

更多请参考[原文链接](http://blog.csdn.net/shinukami/article/details/46369765)

**0x05 维吉利亚加密**

[![那些年做过的ctf之加密篇(补充篇)-安全盒子](http://www.secbox.cn/wp-content/uploads/2015/11/维吉尼亚密码.png)](http://www.secbox.cn/wp-content/uploads/2015/11/维吉尼亚密码.png "那些年做过的ctf之加密篇(补充篇)")

凯撒密码的升级，更多详见[百度百科](http://baike.baidu.com/view/270838.htm)

**0x06 Unicode**

密文样例：

```
\u5927\u5bb6\u597d\uff0c\u6211\u662f\u0040\u65e0\u6240\u4e0d\u80fd\u7684\u9b42\u5927\u4eba\uff01\u8bdd\u8bf4\u5fae\u535a\u7c89\u4e1d\u8fc7\
```

[unicode在线解密](http://tool.chinaz.com/Tools/Unicode.aspx)

**0x07 brainfuck**

类型：

```
++++++++++[>+++++++>++++++++++>+++>+<<<<-]
>++.>+.+++++++..+++.>++.<<+++++++++++++++.
>.+++.------.--------.>+.>.
```

利用BFVM.exe直接解密

用法 loadtxt 1.txt

[brainfuck在线解密](http://www.splitbrain.org/services/ook)

**0x08 摩斯密码**

[![那些年做过的ctf之加密篇(补充篇)-安全盒子](http://www.secbox.cn/wp-content/uploads/2015/11/摩尔密码加密与解密.jpg)](http://www.secbox.cn/wp-content/uploads/2015/11/摩尔密码加密与解密.jpg "那些年做过的ctf之加密篇(补充篇)")

密文样例：--  ---  .-.  ...  .

[摩斯密码在线翻译](http://www.jb51.net/tools/morse.htm)

**0x09 jsfuck or jother**

[把 JavaScript 代码转为 ()[]{}!+ 字符](http://www.freebuf.com/tools/5352.html)

密文样例：

```
[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]][([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]]((![]+[])[+!+[]]+(![]+[])[!+[]+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]+(!![]+[])[+[]]+(![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[!+[]+!+[]+[+[]]]+[+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[!+[]+!+[]+[+[]]])()
```

在线解密：

1.[jsfuck](http://www.jsfuck.com/)

2.[patriciopalladino](http://patriciopalladino.com/files/hieroglyphy/)

其他解密方式：alert(xxx)、console(xxx)、document.write(xxx)即可（xxx为编码内容）。

[![那些年做过的ctf之加密篇(补充篇)-安全盒子](http://www.secbox.cn/wp-content/uploads/2015/11/2.png)](http://www.secbox.cn/wp-content/uploads/2015/11/2.png "那些年做过的ctf之加密篇(补充篇)")

更多jother介绍请参考：[jother编码之谜](http://drops.wooyun.org/web/4410)

**0x0a 培根密码**

[![那些年做过的ctf之加密篇(补充篇)-安全盒子](http://www.secbox.cn/wp-content/uploads/2015/11/培根密码.jpg)](http://www.secbox.cn/wp-content/uploads/2015/11/培根密码.jpg "那些年做过的ctf之加密篇(补充篇)")

培根所用的密码是一种本质上用二进制数设计的。不过，他没有用通常的0和1来表示，而是采用a和b。

详见[百度百科](http://baike.baidu.com/link?url=acaeI3babB7MogPQFh98rDAVSwHfPwh-HnEFTb9cx7DZ5Nz4MkMA14H4SDjBNnOdBsJpliNYa1vnfikQGqvA7K)

**0x0b 猪圈密码又称共济会密码**

[![那些年做过的ctf之加密篇(补充篇)-安全盒子](http://www.secbox.cn/wp-content/uploads/2015/11/猪圈密码加密解密.jpg)](http://www.secbox.cn/wp-content/uploads/2015/11/猪圈密码加密解密.jpg "那些年做过的ctf之加密篇(补充篇)")

详见[百度百科](http://baike.baidu.com/link?url=yN39kWG2pGd9XHo3RjeUAbd7xs0QlnJ2uHzCJfxC03V-fJcQUdfcJ-WuGoAkKGFVE0AxFK4-98wa4FtzvxRA0_)

**0x0c CRC32**

密文样例：4D1FAE0B

```
import zlib
def crc32(st):
crc = zlib.crc32(st)
if crc > 0:
return "%x" % (crc)
else:
return "%x" % (~crc ^ 0xffffffff)
```

更多请参考[原文链接](http://blog.csdn.net/ab748998806/article/details/46382017)

**0x0d其他的一些脑洞加密解密参考：**

[![那些年做过的ctf之加密篇(补充篇)-安全盒子](http://www.secbox.cn/wp-content/uploads/2015/11/当铺密码.jpg)](http://www.secbox.cn/wp-content/uploads/2015/11/当铺密码.jpg "那些年做过的ctf之加密篇(补充篇)")

(1)当铺密码

[![那些年做过的ctf之加密篇(补充篇)-安全盒子](http://www.secbox.cn/wp-content/uploads/2015/11/非斯的象形文字翻译图.png)](http://www.secbox.cn/wp-content/uploads/2015/11/非斯的象形文字翻译图.png "那些年做过的ctf之加密篇(补充篇)")

(2)非斯的象形文字翻译图

[![那些年做过的ctf之加密篇(补充篇)-安全盒子](http://www.secbox.cn/wp-content/uploads/2015/11/ADFGX加密法.png)](http://www.secbox.cn/wp-content/uploads/2015/11/ADFGX加密法.png "那些年做过的ctf之加密篇(补充篇)")

(3)ADFGX加密法

[![那些年做过的ctf之加密篇(补充篇)-安全盒子](http://www.secbox.cn/wp-content/uploads/2015/11/电脑键盘QWE加密法.jpg)](http://www.secbox.cn/wp-content/uploads/2015/11/电脑键盘QWE加密法.jpg "那些年做过的ctf之加密篇(补充篇)")

(4)电脑键盘QWE加密法

[![那些年做过的ctf之加密篇(补充篇)-安全盒子](http://www.secbox.cn/wp-content/uploads/2015/11/电脑键盘棋盘加密.jpg)](http://www.secbox.cn/wp-content/uploads/2015/11/电脑键盘棋盘加密.jpg "那些年做过的ctf之加密篇(补充篇)")

(5)电脑键盘棋盘加密

[![那些年做过的ctf之加密篇(补充篇)-安全盒子](http://www.secbox.cn/wp-content/uploads/2015/11/电脑键盘坐标加密.jpg)](http://www.secbox.cn/wp-content/uploads/2015/11/电脑键盘坐标加密.jpg "那些年做过的ctf之加密篇(补充篇)")

(6)电脑键盘坐标加密

[![那些年做过的ctf之加密篇(补充篇)-安全盒子](http://www.secbox.cn/wp-content/uploads/2015/11/手机键盘加密解密.jpg)](http://www.secbox.cn/wp-content/uploads/2015/11/手机键盘加密解密.jpg "那些年做过的ctf之加密篇(补充篇)")

(7)手机键盘加密解密

[![那些年做过的ctf之加密篇(补充篇)-安全盒子](http://www.secbox.cn/wp-content/uploads/2015/11/数字坐标加密字母.png)](http://www.secbox.cn/wp-content/uploads/2015/11/数字坐标加密字母.png "那些年做过的ctf之加密篇(补充篇)")

(8)数字坐标加密字母

[![那些年做过的ctf之加密篇(补充篇)-安全盒子](http://www.secbox.cn/wp-content/uploads/2015/11/字母表顺序加密法和反字母表加密法和小键盘加密法.jpg)](http://www.secbox.cn/wp-content/uploads/2015/11/字母表顺序加密法和反字母表加密法和小键盘加密法.jpg "那些年做过的ctf之加密篇(补充篇)")

(9)字母表顺序加密法和反字母

分享一个小技巧，如果没有什么思路可以试试[在线词频分析](http://quipqiup.com/index.php)

对于其他一些未知密文，可尝试到下列几个网站转换试试，看看运气

[http://web.chacuo.net/charsetuuencode](http://web.chacuo.net/charsetuuencode)

[http://blog.csdn.net/ab748998806/article/details/46368337](http://blog.csdn.net/ab748998806/article/details/46368337)

说到字符转换，不得不的说到[JPK神器](http://www.wechall.net/applet/JPK_406.jar)

[![那些年做过的ctf之加密篇(补充篇)-安全盒子](http://www.secbox.cn/wp-content/uploads/2015/11/jpk1.png)](http://www.secbox.cn/wp-content/uploads/2015/11/jpk1.png "那些年做过的ctf之加密篇(补充篇)")

在[CTF](http://www.secbox.cn/tag/ctf "浏览关于“CTF”的文章")一次完美优雅的利用请看**AppLeU0**大大的[隐写术总结](http://drops.wooyun.org/tips/4862)中的题目为双图的解题过程，除此之外JPK还有很多功能等你去搞鼓

**0x0e其他加密算法和哈希散列**

AES RSA RC4 Rabbit TripleDes

SHA1 SHA224 SHA256 SHA384 SHA512 MD5 MD4 MD3 MD2 HmacSHA1 HmacSHA224 HmacSHA256 HmacSHA384 HmacSHA512 HmacMD5 PBKDF2 太多了。。。

可以参考网上许多的在线解密网站(百度和谷歌来搜索，小技巧：用谷歌时这样用 MD5 onlie crack, MD5 online decode 类似)和工具。

**0x0f 混淆加密**

列举一些常见的混淆加密

[asp混淆加密](http://www.zhaoyuanma.com/aspfix.html)

[php混淆加密](http://www.zhaoyuanma.com/phpcodefix.html)

[css/js混淆加密](http://tool.css-js.com/)

[VBScript.Encode混淆加密](http://www.zhaoyuanma.com/aspfix.html)