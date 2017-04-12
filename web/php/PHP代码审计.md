# PHP代码审计

## 目录

+ 1. 概述 
+ 2. 输入验证和输出显示 3
    + 2.1 命令注入 4
    + 2.2 跨站脚本 4
    + 2.3 文件包含 5
    + 2.4 代码注入 5
    + 2.5 SQL注入 6
    + 2.6 XPath注入 6
    + 2.7 HTTP响应拆分 6
    + 2.8 文件管理 6
    + 2.9 文件上传 7
    + 2.10 变量覆盖 7
    + 2.11 动态函数 7
+ 3. 会话安全 8
    + 3.1 HTTPOnly设置 8
    + 3.2 domain设置 8
    + 3.3 path设置 8
    + 3.4 cookies持续时间 8
    + 3.5 secure设置 8
    + 3.6 session固定 9
    + 3.7 CSRF 9
+ 4. 加密 9
    + 4.1 明文存储密码 9
    + 4.2 密码弱加密 9
    + 4.3 密码存储在攻击者能访问到的文件 9
+ 5. 认证和授权 10
    + 5.1 用户认证 10
    + 5.2 函数或文件的未认证调用 10
    + 5.3 密码硬编码 10
+ 6. 随机函数 10
    + 6.1 rand() 10
    + 6.2 mt_srand()和mt_rand() 11
+ 7. 特殊字符和多字节编码 11
    + 7.1 多字节编码 11
+ 8. PHP危险函数 11
    + 8.1 缓冲区溢出 11
    + 8.2 session_destroy()删除文件漏洞 12
    + 8.3 unset()-zend_hash_del_key_or_index漏洞 12
+ 9. 信息泄露 13
    + 9.1 phpinfo 13
+ 10. PHP环境 13
    + 10.1 open_basedir设置 13
    + 10.2 allow_url_fopen设置 13
    + 10.3 allow_url_include设置 13
    + 10.4 safe_mode_exec_dir设置 14
    + 10.5 magic_quote_gpc设置 14
    + 10.6 register_globals设置 14
    + 10.7 safe_mode设置 14
    + 10.8 session_use_trans_sid设置 14
    + 10.9 display_errors设置 14
    + 10.10 expose_php设置 14

## 1.概述

代码审核，是对应用程序源代码进行系统性检查的工作。它的目的是为了找到并且修复应用程序在开发阶段存在的一些漏洞或者程序逻辑错误，避免程序漏洞被非法利用给企业带来不必要的风险。

代码审核不是简单的检查代码，审核代码的原因是确保代码能安全的做到对信息和资源进行足够的保护，所以熟悉整个应用程序的业务流程对于控制潜在的风险是非常重要的。审核人员可以使用类似下面的问题对开发者进行访谈，来收集应用程序信息。

应用程序中包含什么类型的敏感信息，应用程序怎么保护这些信息的？

应用程序是对内提供服务，还是对外？哪些人会使用，他们都是可信用户么？

应用程序部署在哪里？

应用程序对于企业的重要性？

最好的方式是做一个checklist，让开发人员填写。Checklist能比较直观的反映应用程序的信息和开发人员所做的编码安全，它应该涵盖可能存在严重漏洞的模块，例如：数据验证、身份认证、会话管理、授权、加密、错误处理、日志、安全配置、网络架构。

## 2.输入验证和输出显示

大多数漏洞的形成原因主要都是未对输入数据进行安全验证或对输出数据未经过安全处理，比较严格的数据验证方式为：

1. 对数据进行精确匹配
2. 接受白名单的数据
3. 拒绝黑名单的数据
4. 对匹配黑名单的数据进行编码

在PHP中可由用户输入的变量列表如下：

+ $_SERVER
+ $_GET
+ $_POST
+ $_COOKIE
+ $_REQUEST
+ $_FILES
+ $_ENV
+ $_HTTP_COOKIE_VARS
+ $_HTTP_ENV_VARS
+ $_HTTP_GET_VARS
+ $_HTTP_POST_FILES
+ $_HTTP_POST_VARS
+ $_HTTP_SERVER_VARS

我们应该对这些输入变量进行检查

### 2.1 命令注入

PHP执行系统命令可以使用以下几个函数：system、exec、passthru、“、shell_exec、popen、proc_open、pcntl_exec

我们通过在全部程序文件中搜索这些函数，确定函数的参数是否会因为外部提交而改变，检查这些参数是否有经过安全处理。

防范方法：

1. 使用自定义函数或函数库来替代外部命令的功能
2. 使用escapeshellarg函数来处理命令参数
3. 使用safe_mode_exec_dir指定可执行文件的路径

### 2.2 跨站脚本

反射型跨站常常出现在用户提交的变量接受以后经过处理，直接输出显示给客户端；存储型跨站常常出现在用户提交的变量接受过经过处理后，存储在数据库里，然后又从数据库中读取到此信息输出到客户端。输出函数经常使用：echo、print、 printf、vprintf、< %=$test%>

对于反射型跨站，因为是立即输出显示给客户端，所以应该在当前的php页面检查变量被客户提交之后有无立即显示，在这个过程中变量是否有经过安全检查。

对于存储型跨站，检查变量在输入后入库，又输出显示的这个过程中，变量是否有经过安全检查。

防范方法：

1. 如果输入数据只包含字母和数字，那么任何特殊字符都应当阻止
2. 对输入的数据经行严格匹配，比如邮件格式，用户名只包含英文或者中文、下划线、连字符
3. 对输出进行HTML编码，编码规范

```
< < > >
( (
) )
# #
& &
” “
‘ ‘
` %60
```

### 2.3 文件包含

PHP可能出现文件包含的函数：include、include_once、require、require_once、show_source、highlight_file、readfile、file_get_contents、fopen、file

防范方法：

1. 对输入数据进行精确匹配，比如根据变量的值确定语言en.php、cn.php，那么这两个文件放在同一个目录下’language/’.$_POST[‘lang’].’.php’，那么检查提交的数据是否是en或者cn是最严格的，检查是否只包含字母也不错
2. 通过过滤参数中的/、..等字符

### 2.4 代码注入

PHP可能出现代码注入的函数：eval、preg_replace+/e、assert、call_user_func、call_user_func_array、create_function

查找程序中程序中使用这些函数的地方，检查提交变量是否用户可控，有无做输入验证

防范方法：

1. 输入数据精确匹配
2. 白名单方式过滤可执行的函数

### 2.5 SQL注入

SQL注入因为要操作数据库，所以一般会查找SQL语句关键字：insert、delete、update、select，查看传递的变量参数是否用户可控制，有无做过安全处理

防范方法：

使用参数化查询

### 2.6 XPath注入

Xpath用于操作xml，我们通过搜索xpath来分析，提交给xpath函数的参数是否有经过安全处理

防范方法：

对于数据进行精确匹配

### 2.7 HTTP响应拆分

PHP中可导致HTTP响应拆分的情况为：使用header函数和使用$_SERVER变量。注意PHP的高版本会禁止HTTP表头中出现换行字符，这类可以直接跳过本测试。

防范方法：

1. 精确匹配输入数据
2. 检测输入输入中如果有\r或\n，直接拒绝

### 2.8 文件管理

PHP的用于文件管理的函数，如果输入变量可由用户提交，程序中也没有做数据验证，可能成为高危漏洞。我们应该在程序中搜索如下函数：copy、rmdir、unlink、delete、fwrite、chmod、fgetc、 fgetcsv、fgets、fgetss、file、file_get_contents、fread、readfile、ftruncate、 file_put_contents、fputcsv、fputs，但通常PHP中每一个文件操作函数都可能是危险的。

http://ir.php.net/manual/en/ref.filesystem.php

防范方法：

1. 对提交数据进行严格匹配
2. 限定文件可操作的目录

### 2.9 文件上传

PHP文件上传通常会使用move_uploaded_file，也可以找到文件上传的程序进行具体分析

防范方式：

1. 使用白名单方式检测文件后缀
2. 上传之后按时间能算法生成文件名称
3. 上传目录脚本文件不可执行
4. 注意%00截断

### 2.10 变量覆盖

PHP变量覆盖会出现在下面几种情况：

1.遍历初始化变量

例：

```
foreach($_GET as $key => $value)
$$key = $value;
```

2.函数覆盖变量：parse_str、mb_parse_str、import_request_variables

3.Register_globals=ON时，GET方式提交变量会直接覆盖

防范方法：

1. 设置Register_globals=OFF
2. 不要使用这些函数来获取变量

### 2.11 动态函数

当使用动态函数时，如果用户对变量可控，则可导致攻击者执行任意函数。

例：

```
< ?php $myfunc = $_GET['myfunc']; $myfunc(); ?>
```

防御方法：

不要这样使用函数

## 3.会话安全

### 3.1 HTTPOnly设置

session.cookie_httponly = ON时，客户端脚本(JavaScript等)无法访问该cookie，打开该指令可以有效预防通过XSS攻击劫持会话ID

### 3.2 domain设置

检查session.cookie_domain是否只包含本域，如果是父域，则其他子域能够获取本域的cookies

### 3.3 path设置

检查session.cookie_path，如果网站本身应用在/app，则path必须设置为/app/，才能保证安全

### 3.4 cookies持续时间

检查session.cookie_lifetime，如果时间设置过程过长，即使用户关闭浏览器，攻击者也会危害到帐户安全

### 3.5 secure设置

如果使用HTTPS，那么应该设置session.cookie_secure=ON，确保使用HTTPS来传输cookies

### 3.6 session固定

如果当权限级别改变时（例如核实用户名和密码后，普通用户提升到管理员），我们就应该修改即将重新生成的会话ID，否则程序会面临会话固定攻击的风险。

### 3.7 CSRF

跨站请求伪造攻击，是攻击者伪造一个恶意请求链接，通过各种方式让正常用户访问后，会以用户的身份执行这些恶意的请求。我们应该对比较重要的程序模块，比如修改用户密码，添加用户的功能进行审查，检查有无使用一次性令牌防御csrf攻击。

## 4.加密

### 4.1 明文存储密码

采用明文的形式存储密码会严重威胁到用户、应用程序、系统安全。

### 4.2 密码弱加密

使用容易破解的加密算法，MD5加密已经部分可以利用md5破解网站来破解

### 4.3 密码存储在攻击者能访问到的文件

例如：保存密码在txt、ini、conf、inc、xml等文件中，或者直接写在HTML注释中

## 5.认证和授权

### 5.1 用户认证

检查代码进行用户认证的位置，是否能够绕过认证，例如：登录代码可能存在表单注入。

检查登录代码有无使用验证码等，防止暴力破解的手段

### 5.2 函数或文件的未认证调用

一些管理页面是禁止普通用户访问的，有时开发者会忘记对这些文件进行权限验证，导致漏洞发生

某些页面使用参数调用功能，没有经过权限验证，比如index.php?action=upload

### 5.3 密码硬编码

有的程序会把数据库链接账号和密码，直接写到数据库链接函数中。

## 6.随机函数

### 6.1 rand()

rand()最大随机数是32767，当使用rand处理session时，攻击者很容易破解出session，建议使用mt_rand()

### 6.2 mt_srand()和mt_rand()

PHP4和PHP5<5.2.6，这两个函数处理数据是不安全的。在web应用中很多使用mt_rand来处理随机的session，比如密码找回功能等，这样的后果就是被攻击者恶意利用直接修改密码。

## 7.特殊字符和多字节编码

### 7.1 多字节编码

## 8.PHP危险函数

### 8.1 缓冲区溢出

confirm_phpdoc_compiled

影响版本：

phpDocumentor phpDocumentor 1.3.1

phpDocumentor phpDocumentor 1.3 RC4

phpDocumentor phpDocumentor 1.3 RC3

phpDocumentor phpDocumentor 1.2.3

phpDocumentor phpDocumentor 1.2.2

phpDocumentor phpDocumentor 1.2.1

phpDocumentor phpDocumentor 1.2

mssql_pconnect/mssql_connect

影响版本：PHP < = 4.4.6

crack_opendict

影响版本：PHP = 4.4.6

snmpget

影响版本：PHP <= 5.2.3

ibase_connect

影响版本：PHP = 4.4.6

unserialize

影响版本：PHP 5.0.2、PHP 5.0.1、PHP 5.0.0、PHP 4.3.9、PHP 4.3.8、PHP 4.3.7、PHP 4.3.6、PHP 4.3.3、PHP 4.3.2、PHP 4.3.1、PHP 4.3.0、PHP 4.2.3、PHP 4.2.2、PHP 4.2.1、PHP 4.2.0、PHP 4.2-dev、PHP 4.1.2、PHP 4.1.1、PHP 4.1.0、PHP 4.1、PHP 4.0.7、PHP 4.0.6、PHP 4.0.5、PHP 4.0.4、PHP 4.0.3pl1、PHP 4.0.3、PHP 4.0.2、PHP 4.0.1pl2、PHP 4.0.1pl1、PHP 4.0.1

### 8.2 session_destroy()删除文件漏洞

影响版本：不祥，需要具体测试

测试代码如下：


```
01	<?php
02	session_save_path(‘./’);
03	session_start();
04	if($_GET[‘del’]) {
05	session_unset();
06	session_destroy();
07	}else{
08	$_SESSION[‘do’]=1;
09	echo(session_id());
10	print_r($_SESSION);
11	}
12	?>
```

当我们提交cookieHPSESSIONID=/../1.php，相当于删除了此文件

### 8.3 unset()-zend_hash_del_key_or_index漏洞

zend_hash_del_key_or_index PHP4小于4.4.3和PHP5小于5.1.3，可能会导致zend_hash_del删除了错误的元素。当PHP的unset()函数被调用时，它会阻止变量被unset。

## 9.信息泄露

### 9.1 phpinfo

如果攻击者可以浏览到程序中调用phpinfo显示的环境信息，会为进一步攻击提供便利

## 10.PHP环境

### 10.1 open_basedir设置

open_basedir能限制应用程序能访问的目录，检查有没有对open_basedir进行设置，当然有的通过web服务器来设置，例如：apache的php_admin_value，nginx+fcgi通过conf来控制php设置

### 10.2 allow_url_fopen设置

如果allow_url_fopen=ON，那么php可以读取远程文件进行操作，这个容易被攻击者利用

### 10.3 allow_url_include设置

如果allow_url_include=ON，那么php可以包含远程文件，会导致严重漏洞

### 10.4 safe_mode_exec_dir设置

这个选项能控制php可调用的外部命令的目录，如果PHP程序中有调用外部命令，那么指定外部命令的目录，能控制程序的风险

### 10.5 magic_quote_gpc设置

这个选项能转义提交给参数中的特殊字符，建议设置magic_quote_gpc=ON

### 10.6 register_globals设置

开启这个选项，将导致php对所有外部提交的变量注册为全局变量，后果相当严重

### 10.7 safe_mode设置

safe_mode是PHP的重要安全特性，建议开启

### 10.8 session_use_trans_sid设置

如果启用 session.use_trans_sid，会导致 PHP 通过 URL 传递会话 ID，这样一来，攻击者就更容易劫持当前会话，或者欺骗用户使用已被攻击者控制的现有会话。

### 10.9 display_errors设置

如果启用此选项，PHP将输出所有的错误或警告信息，攻击者能利用这些信息获取web根路径等敏感信息

### 10.10 expose_php设置

如果启用 expose_php 选项，那么由 PHP 解释器生成的每个响应都会包含主机系统上所安装的 PHP 版本。了解到远程服务器上运行的 PHP 版本后，攻击者就能针对系统枚举已知的盗取手段，从而大大增加成功发动攻击的机会。

## 参考文档：

https://www.fortify.com/vulncat/zh_CN/vulncat/index.html

http://secinn.appspot.com/pstzine/read?issue=3&articleid=6

http://riusksk.blogbus.com/logs/51538334.html

http://www.owasp.org/index.php/Category:OWASP_Code_Review_Project

《来文来源于DoDo》