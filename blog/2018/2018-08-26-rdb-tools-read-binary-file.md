---
layout: post
title: "［深入理解Redis］读取RDB文件"
date: '2018-08-27'
tags: tech
author: hoohack
categories: Redis
excerpt: 'redis,redis源码,redis 4.0源码,RDB文件,读取二进制文件,Golang读取文件'
keywords: 'redis,redis源码,redis 4.0源码,RDB文件,读取二进制文件,Golang读取文件'
---

最近在做一个解析rdb文件的功能，途中遇到了一些问题，也解决了一些问题。具体为什么要做这件事情之后再详谈，本次主要想聊聊遇到的开始处理文件时遇到的第一个难题：理解RDB文件的协议、如何读取二进制文件。

## RDB文件
[［Redis源码阅读］redis持久化](https://www.hoohack.me/2018/04/04/deep-learning-redis-durability)
文章介绍过，Redis的持久化是通过RDB和AOF实现的。Redis的RDB文件是二进制格式的文件，从这个方面再次体现了Redis是基于内存的缓存数据库，不管对于存储到硬盘还是恢复数据都十分快捷。Redis有多种数据类型：string、list、hash、set、zset，不同数据类型占用的内存大小是不一样的，解析出自然语言可以识别的数据就需要使用不同的方法，保存到文件的时候也需要一些协议或者规则。这有点类似于编程语言里面的数据类型，不同的数据类型占用的字节大小不一致，但是保存到计算机都是二进制的编码，就看是读取多少个字节，以怎样的方式解读。



举个例子，redis的对象类型是特定的几个字符表示，0代表字符串，读取到字符串类型后，紧接着就是字符串的长度，保存着接下来需要读取的字节大小，读取到的字节最终构成完整字符串对象的值。对于保存了`"name" => "hoohack"`键值对的字符串对象保存到内存可以用下图表示：

![redis字符串存储](https://www.hoohack.me/assets/images/2018/08/redis-string-storage.png)

当然，除了字符串，redis还有列表，集合，哈希等各种对象，针对这些类型，在RDB文件里面都有不同的规则定义，只需要按照RDB文件格式的协议来解读文件，就能完整无误地把文件解读成自然语言能描述的字符。

仔细对比，可以发现跟计算机的操作方式是类似的，数据保存在计算机都是二进制的，具体的值应该看需要多少个字节，以什么类型解析，读取不同的字节解析到的值是不一样的，同样的字节大小，但是使用不同类型保存，只要做适当的转换，也是正确的。比如在C语言中的`void *`指针是4个字节，int也是4个字节，定义一个int整数，将它保存到`void *`也是没问题的，读取的时候只需要做一次类型转换就可以了。

因此，解读RDB文件最关键的就是理解RDB文件的协议，只要理解完RDB文件格式的协议，根据定义好的协议来解析各种数据类型的数据。更详细的RDB文件协议可以参考RDB文件格式的定义文档：[RDB file format](http://rdb.fnordig.de/file_format.html)。

## 查看RDB文件
先清空redis数据库，保存一个键值对，然后执行save命令将当前数据库的数据保存的rdb文件，得到文件dump.rdb。

    127.0.0.1:6379> flushall
    OK
    127.0.0.1:6379> set name hoohack
    OK
    127.0.0.1:6379> save
    OK

### cat查看文件：
![cat-rdb-file](https://www.hoohack.me/assets/images/2018/08/cat-rdb-file.png)

可以看到是一个包含乱码的文件，因为文件是以二进制的格式保存，要想打印出人类能看出的语言，可以通过linux的od命令查看。od命令用于输出文件的八进制、十六进制或其他格式编码的字节，通常用于输出文件中不能直接显示在终端的字符，注意输出的是字节，分隔符之间的字符都是保存在一个字节的。

### od命令输出文件八进制、十六进制等格式
通过man手册可以看到，打印出16进制格式的参数是x，字符是c，将字符与十六进制的对应关系打印出来：`od -A x -t x1c -v dump.rdb`

打印得出结果如下：

![linux-od](https://www.hoohack.me/assets/images/2018/08/linux-od.png)

从上图看到，文件打印出来的都是一些十六进制的数字，转换成十进制再去ASCII码表就能查找到对应的字符。比如第一个字符，52=5*16+2*1=82='R’。
在这里说一句，个人觉得这od命令非常有用，在解析数据出现疑惑的时候，就是通过这个命令排查遇到的问题。把文件内容打印出来，找到当前读取的字符，对比RDB文件协议，看看究竟要解析的是什么数据，再对比代码中的解析逻辑，看看是否有问题，然后再修正代码。
如果觉得敲命令麻烦，可以把文件上传然后在线查看：[https://www.onlinehexeditor.com](https://www.onlinehexeditor.com)

## 数据的二进制保存和读取
我们知道，计算机的所有数据都是以二进制的格式保存的，我们看到的字符是通过读取二进制然后解析出来的数据，程序根据不同数据类型做相应的转换，然后展示出来的就是我们看到的字符。
计算机允许多种数据类型，比如有：32位整数、64位整数、字符串、浮点数、布尔值等等，不同的数据类型是通过读取不同大小的字节，根据类型指定的读取方式读取出来，就是想要的数据了。

解析数据的第一步，就是读取数据。在计算机里面，大家所知道的数据都是逐个字节地读取数据。因此，首先要实现的就是按照字节去读取文件。

本次采用实现的解析RDB文件功能的语言是Golang，在Golang的文件操作的API里，提供了按字节读取的函数`File.ReadAt`。

函数原型如下：
    
    func (f *File) ReadAt(b []byte, off int64) (n int, err error)

函数从File指针的指向的位置off开始，读取len(b)个字节的数据，并保存到b中。
根据对API的理解，自己实现了一个按照字节读取文件数据的函数，函数接收长度值，代表需要读取的字节长度。具体实现代码如下：

    type Rdb struct {
        fp *os.File
        ... // other field
    }

    func (r *Rdb) ReadBuf(length int64) ([]byte, error) {
        // 初始化一个大小为length的字节数组
        buf := make([]byte, length)

        // 从curIndex开始读取length个字节
        size, err := r.fp.ReadAt(buf[:length], r.curIndex)

        checkErr(err)

        if size < 0 {
            fmt.Fprintf(os.Stderr, "cat: error reading: %s\n", err.Error())
            return []byte{}, err
        } else {
            // 读取成功，更新文件操作的偏移量
            r.curIndex += length
            return buf, nil
        }
    }

### Golang的数据转换
上面实现的函数返回的是字节数组，当函数返回读取到的数据后，如果需要保存在不同的数据类型就需要做转换，Golang也提供了比较强大的api。以下是我在解析数据时遇到的数据类型转换的解决方案，希望对大家有帮助。
字符串
    
    str := string(buf)

int整数，先转为二进制的值，然后再用int32类型格式化
    
    intVal := int(binary.BigEndian.Uint32(buf))

int64整数，先转为二进制的值，然后再用int64类型格式化
    
    int64Val := int64(int16(binary.LittleEndian.Uint16(valBuf)))

浮点数
    
    floatVal, err := strconv.ParseFloat(string(floatBuf), 64)

float64浮点数，先转为二进制的值，再调用math库的Float64frombits函数转换二进制的值为float64类型
    
    floatBit := binary.LittleEndian.Uint64(buf)
    floatVal := math.Float64frombits(floatBit)

## 总结
理论上的理解和实践上的应用是不一样的，虽然大家都知道数据是二进制的，就是怎么怎么解析，但是真正实现起来还是不少问题。通过操作二进制文件的一次实践，收获了以下几点：

1、更深刻地理解到数据在计算机中的保存方式，一切都是0和1的二进制内容，只是看你要怎么用而已，适当的类似转换也可以得到你想要的内容

2、在某个系统下操作就要遵循已定义好的协议，不然得到的都是乱套或者乱码的东西，比如数据的字节序不同也会使数据的解析结果不一致

原创文章，文笔有限，才疏学浅，文中若有不正之处，万望告知。

如果本文对你有帮助，请点个赞吧，谢谢^_^



