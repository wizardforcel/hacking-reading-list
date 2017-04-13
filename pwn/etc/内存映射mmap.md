# 内存映射mmap

> 来源：http://www.linuxidc.com/Linux/2016-12/137775.htm

### Table of Contents

*   1\. 什么是mmap
*   2\. 使用方法
    *   2.1\. mmap构造器的格式
    *   2.2\. 例子1
    *   2.3\. 例子2
*   3\. 其它
*   4\. 参考资料

## 什么是mmap

通常在Unix系统里有两种操作的数据类型：内存地址和流文件(stream)。通过操作内存地址的方法涉及的操作有:pointers, malloc/free之类，而操作流文件涉及的方法有read/write/seek等系统调用或者send/recv/etc等socket操作。而mmap提供了结合上述两种类型的操作方式。简单来讲，mmap可以创建一个内存映射(memory-mapped)类型的文件，可以直接在内存操作文件，而不需要使用通常的read,wirte这些系统I/O调用。这样的好处是避免了操作文件是频繁地系统调用。

## 使用方法

内存映射(memory-mapped)可以像字符串和文件对象一样操作，通过 `mmap` 来创建。

例子中采用的hello.txt文件如下:

```
Hello, i am Nisen,
Nice to meet you!
Goodbye.
```

### mmap构造器的格式

```
# Unix version
class mmap.mmap(fileno, length[, flags[, prot[, access[, offset]]]])

# Windows version
class mmap.mmap(fileno, length[, tagname[, access[, offset]]])
```

fileno是流文件的描述符,length指定映射文件到内存的bytes的长度，设置为0的话代表全部。Unix接口中的flags指定这个创建出来的mapping是否对创建的进程私有，默认是共享的。prot和access指定需要的内存保护（读写相关），其它参数的含义可以参照 [文档](https://docs.python.org/2.7/library/mmap.html) 。

接下来让我们采用Unix的接口，做些实验吧。

### 例子1

```
import mmap

with open('hello.txt', 'r') as f:
    m = mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ)
    print m.readline()
    m.close()
```

运行的结果如下:

```
Hello, i am Nisen,
```

python3.2以后mmap支持用with的方式操作

```
# New in version 3.2: Context manager support.
with open('hello.txt', 'r') as f:
    with mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ) as m:
        print('First 10 bytes via read :', m.read(10))
        print('First 10 bytes via slice:', m[:10])
```

运行后的结果

```
python3 test.py
First 10 bytes via read : b'Hello, i am Nisen,\nNice to meet you!\nGoodbye.\n'
First 10 bytes via slice: b'Hello, i a'
```

### 例子2

常见的方法如下

```
with open('hello.txt', 'r+') as f:
    # 指定访问权限为write, 一共有3种权限指定:ACCESS_READ, ACCESS_WRITE, ACCESS_COPY
    m = mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_WRITE)

    # 输出一行
    print m.readline()

    # 指针重置
    m.seek(0)

    # 查找"Nisen"出现的第一个地方，返回索引
    index = m.find('Nisen')
    print index

    m.seek(0)

    # 直接修改内容
    m[index: index+5] = "Rubby"

    # 将内存中的修改存到磁盘中的文件上
    m.flush()

    m.seek(0)
    print m.readline()

    # 关闭内存映射文件
    m.close()
```

运行结果如下：

```
➜ python test2.py
Hello, i am Nisen,

12
Hello, i am Rubby,
```

## 其它

1.  mmap的read()方法在python3.3开始可以接受空参数，表示读取文件所有的内容
2.  在创建mmap对象指定权限的时候，注意本来文件描述符拥有的权限。如果使用open()打开文件的权限指定了'r', 用mmap创建映射对象时指定 `ACCESS_WRITE` ，那么会报 `Permission denied` 的错误
3.  关于文件打开模式"r+"和"w+"的用法可以参考这里 [这里](http://stackoverflow.com/questions/21113919/difference-between-r-and-w-in-fopen)
4.  在多线程编程时，如果多个线程以只读的方式访问同一个文件，那么可以采用mmap创一个映射对象来减少内存的使用提升性能
5.  mmap会将文件对象一次读取到连续内存空间上，如果文件过大导致找不到可用的内存空间，那么创建这个映射对象将会失败
6.  mmap加快文件操作的例子可以参照 [这里](http://pythoncentral.io/memory-mapped-mmap-file-support-in-python/)

## 参考资料

*   https://docs.python.org/2.7/library/mmap.html
*   https://docs.python.org/3.5/library/mmap.html
*   https://pymotw.com/3/mmap/
*   http://pythoncentral.io/memory-mapped-mmap-file-support-in-python/
*   http://stackoverflow.com/questions/21113919/difference-between-r-and-w-in-fopen
*   https://blog.schmichael.com/2011/05/15/sharing-python-data-between-processes-using-mmap/
