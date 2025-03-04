---
layout: post
title:  "Linux命令chmod学习"
date:   '2015-04-22'
tags: tech
author: hoohack
categories: Linux
excerpt: 'Linux,chmod,修改权限'
keywords: 'Linux,chmod,修改权限'
---

chmod命令用得很多，但是有时会忘记此命令的正确用法和一些注意事项。最近用得比较多，总结一下。

##chmod命令用途
用于改变Linux系统的文件的访问权限。通常用它来控制文件的访问权限，使文件可写或者使文件只允许某些用户进入。

##Linux系统文件权限介绍
在Linux系统中，一切都是文件。Linux系统中的每个文件都有访问许可权限，用来确定各种用户可以通过哪种访问方式对文件录进行访问和操作。
文件的访问权限分为只读只写和可执行三种。



>只读权限表示只允许读取其内容，禁止对其做任何的其他操作。可执行权限表示

>可执行表示允许将该文件作为一个程序执行

>可写权限表示可以对文件进行写操作(修改或增加)

##操作文件用户的不同类型
>owner 文件所有者
>group 同组用户
>other 其他用户

每一个文件的访问权限都有三组。每组用三位表示，分别为文件所有者的读、写和执行权限；与文件所有者同组的用户的读、写和执行权限；系统中其他用户的读、写和执行权限。如果需要查看文件权限的详细信息时，可以使用`ls -l`命令。例如：

![ls-l](http://7u2eqw.com1.z0.glb.clouddn.com/ls-lcommand.png)

确定了一个文件的访问权限后，可以利用Linux系统提供的chmod命令来给文件重新设定不同的访问权限。

##命令格式
    
    chmod [-cfvR] [--help] [--version] mode file

###参数说明
- -c 当发生改变时，报告处理信息
- -f 错误信息不输出
- -R 处理指定目录以及其子目录下的所有文件
- -v 运行时显示详细处理信息

###权限范围代号
u ：目录或者文件的当前的用户
g ：目录或者文件的当前的群组
o ：除了目录或者文件的当前用户或群组之外的用户或者群组
a ：所有的用户及群组

###权限代号：
r ：读权限，用数字4表示
w ：写权限，用数字2表示
x ：执行权限，用数字1表示
- ：删除权限，用数字0表示
s ：特殊权限 

##chmod命令用法
此命令有两种用法

###文字设定法
>使用字母和操作符表达式。如

    chmod a+x phptest.log #给所有用户添加可执行此文件的权限

###数字设定法
数字表示的属性的含义：0表示没有权限，1表示可执行权限，2表示可写权限，4表示可读权限，然后将其相加。数字属性是3歌0-7的八进制数，对应的用户是u、g、o。

>使用数字改变文件或目录的权限。如

    chmod 777 phptest.log #使所有用户可读可写可执行该文件
    


##使用实例
###实例1： 增加文件所有用户组可执行权限

    chmod a+x tmp.log

###实例2：同时修改不同用户权限
    
    chmod ug+w,o-x log2015.log

###实例3：删除文件权限
    
    chmod a-x log2015.log
    
