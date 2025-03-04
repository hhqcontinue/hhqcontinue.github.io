---
layout: post
title: "［从0到1编写服务器］准备知识"
date: '2018-09-05'
tags: tech
author: hoohack
categories: Network
excerpt: '网络编程,服务器编程,LinuxC,socket,tcp/ip'
keywords: '网络编程,服务器编程,LinuxC,socket,tcp/ip'
---

### 前言
在互联网发达的时代里，我们的开发过程中，很多场景几乎都需要跟网络打交道，数据库连接、redis连接、nginx转发、RPC服务等等，这个服务器软件的底层实现本质都是网络编程，这也是为什么很多公司在面试的时候都会问到计算机网络。诚然，在正常开发的过程下，几乎不会去自己编写一个完整的服务器，但是，在开发中理解一些概念性的知识却非常有用，甚至在排查一下稀奇古怪的网络错误的时候，TCP/IP协议可以发挥巨大的用处。如果在排查奇怪的问题时，你把学到的网络知识发挥了用途，那么你在公司的前景就不用多说了，而且，作为一个有追求的程序员，不能仅仅满足于业务开发，底层的计算机原理也要经常巩固。毕竟基础知识真的非常重要。



> 这里强调一次，不是说学了网络编程就要去做服务器开发，而是了解网络编程的基础，会有利于提高平时开发调试问题的能力以及理解问题的深度。

学习网络编程是很有必要的，现在市面上有很多书籍和博客，大家也读了很多，But，程序员都是以动手为主，不能仅限于理论知识，需要动手实践来体验当中的一些坑或者难点，解决遇到的难点和坑会使你成长一大步。在我开来，指导别人需要在自己懂的前提下，去告诉别人是怎么做的，通过一个最简单的demo来指导，然后再让他人去深入，而不是一股脑地为对方注入一些理论上的知识。

### 网络交互的本质

>  抛开一切术语来说，其实网络交互的本质就是一次通信。



对于旧时代人而言，一开始的通信方式是写信、飞鸽传书，之后就是电话，而服务器之间的通信就通过网络来进行，而在通信过程中，最需要掌握的就是如何建立连接，然后确保连接成功，如何在网络中传递数据，数据传递的格式是怎样的（协议），等等。

网络通信要解决的首要问题是如何唯一地标示一个通信的主体，比如网络上的两台服务器，如果无法唯一地标示两台服务器，那么就无法建立唯一的连接，通信自然也就无从谈起；再比如网络上的两个服务：两个用户使用微信通信，如果不能唯一标示两个微信客户端应用，A想发给B的消息却发送到C了，那么用户的通信就会变得混乱。

![通信](https://www.hoohack.me/assets/images/2018/09/TCP-Communicate.png)



在机器上，我们通过进程ID来唯一标识一个进程，在网络中，ip地址唯一标识了网站中的一台计算机，而服务的端口号则唯一标识了计算机上的一个应用服务，IP地址+端口则能唯一标识网络中通信的唯一一个通信主体。



![唯一标识通信载体](https://www.hoohack.me/assets/images/2018/09/mark-communicate.png)



另一个问题就是通信的实现，通过什么样的方式、使用什么协议可以完成一次通信呢？这就是TCP/IP做的事情，要了解通信的实现，就需要学习socket以及它的相关api。

### socket介绍

socket直译的意思就是网络套接字， 是一种操作系统提供的进程间通信机制。下面是从维基百科参考的解释：


> 在操作系统中，通常会为应用程序提供一组应用程序接口（API），称为套接字接口（英语：socket API）。应用程序可以通过套接字接口，来使用网络套接字，以进行数据交换。最早的套接字接口来自于4.2 BSD，因此现代常见的套接字接口大多源自Berkeley套接字（Berkeley sockets）标准。在套接字接口中，以IP地址及通信端口组成套接字地址（socket address）。远程的套接字地址，以及本地的套接字地址完成连线后，再加上使用的协议（protocol），这个五元组（five-element tuple），作为套接字对（socket pairs），之后就可以彼此交换数据。例如，在同一台计算机上，TCP协议与UDP协议可以同时使用相同的port而互不干扰。 操作系统根据套接字地址，可以决定应该将数据送达特定的进程或线程。这就像是电话系统中，以电话号码加上分机号码，来决定通话对象一般。

个人的理解，socket就是操作系统用于定义操作计算机网络通信的一个媒介。

### socketAPI

对于一次TCP连接而言，建立连接主要通过socket、bind、listen、accept这些API，整个流程如下图所示：

![TCP连接](https://www.hoohack.me/assets/images/2018/09/TCP-Connect.png)

下面简单介绍下这些API的用法和函数参数的含义，更详细的操作可以直接读[man page文档](http://man7.org)。

#### socket（创建一个用于通信的节点）

```c
int socket(int domain, int type, int protocol);
```



**参数**

domain：指定通信的域，选择通信的协议族，比较常见的有：AF_INET（代表ipv4）、AF_INET6（代表ipv6）等等，这个参数指定了ip地址的格式。

type：指定通信的方式，传递数据的方式，比较常见的有：SOCK_STREAM、SOCK_DGRAM。

protocol：指定通信的协议，常见的有IPPROTO_TCP、IPPROTO_UDP、IPPRO_SCTP。

**返回值：int**

函数的返回值是文件描述符，该文件描述符是一个正整数，唯一标识服务端与某客户端的连接，服务端和客户端可以通过此连接进行通信。出错情况下，会返回-1，并设置errno，可以通过errno获得出错信息。

#### bind（将生成的文件描述符绑定到需要监听的端口）

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```



**参数**

sockfd：通过socket函数返回的文件描述符

addr：指定需要监听的地址，地址包含了ip和端口，该结构体定义如下：

```c
struct sockaddr {
    sa_family_t sa_family;
    char sa_data[14];
}
```


实际上，在调用bind函数时传递的参数定义是struct sockaddr_in，定义如下：

```c
struct sockaddr_in {
    short int sin_family; /* 协议族，在socket编程中基本为AF_INET */
    unsigned short int sin_port; /* 端口号r */
    struct in_addr sin_addr; /* IP地址 */
    unsigned char sin_zero[8]; /* 空字节，为了让sockaddr和sockaddr_in有相同的字节大小 */
};
```


sockaddr_in结构体清楚定义了ip地址和端口，而socketaddr结构体则是将ip地址和端口号捆绑在一起保存在data了，因此使用sockaddr_in进行初始化可以分开存储ip地址和结构体，也让代码看起来更加清晰，之后可以通过类型转换传递到bind函数。

addrlen：第二个参数的大小。

**返回值：int**

成功返回0，出错情况下，会返回-1，并设置errno，可以通过errno获得出错信息。

#### listen（服务端侧使用，开始监听已经建立完连接的socket）

```c
int listen(int sockfd, int backlog)
```



**参数**

sockfd：指向socket的文件描述符

backlog：指定请求数量缓冲区长度，如果请求数量超过backlog的大小，客户端会收到报错信息。

**返回值：int**

成功返回0，出错情况下，会返回-1，并设置errno，可以通过errno获得出错信息。

#### connect（客户端请求连接服务端）

```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t socklen);
```


参数含义与bind相同。

**返回值：int**

成功返回0，出错情况下，会返回-1，并设置errno，可以通过errno获得出错信息。

#### accept（接收请求的到来）

int accept(int sockfd, struct sockaddr *addr, socklen_t *socklen);

accept参数的含义与bind类似，有一个不一样的是socklen，是一个引用传递方式的参数，调用方需要默认的长度，即参数addr的长度，函数调用成功后，会将真实的大小写到socklen。

#### close（关闭打开的文件描述符）

```c
int close(int fd);
```


**参数**

fd：文件描述符。

### linux I/O API

建立连接后，就需要开始传递数据进行通信，比较常用的I/O api有read、write、recv、send。

#### read（从文件描述符读取count字节的数据，并保存到buf）

```c
ssize_t read(int fd, void *buf, size_t count);
```

**参数**
fd：已经打开的文件描述符

buf：保存读取数据的指针

count：要读取的数据字节大小，不能为0

返回值 int

如果成功，返回已经读取的字节大小，0表示到达文件结尾，-1表示错误。

#### write（写入在buf中count个字节的数据到打开的文件描述符fd中）

```c
ssize_t write(int fd, const void *buf, size_t count);
```

**参数**
fd：已经打开的文件描述符

buf：保存数据的指针

count：要写入的数据字节大小

返回值：如果成功，返回写入的字节大小，返回的值比count小不算一种错误，如果出错，返回-1

### 语言基础

本次的demo选用的都是C语言，无可否认，使用C语言编写服务器真的特别麻烦。但是个人比较喜欢C语言（虽然C语言比较水，但就是喜欢它），觉得它是一门特别优秀的语言，而且C语言最接近计算机底层的操作，程序出现各种错误和内存操作都需要程序员去考虑，涉及到的内存细节也更多，这些情况在更高级比如Java、Go这些语言里是很少能见到的，这样一来，除了能通过此练习巩固编程思维之外，还能了解到高级语言的API是如何封装的，怎么考虑异常情况等等，如果是自己设计的话，会怎么编写出强壮且维护性强的API。因此，学习服务器编程，需要准备一些C语言的基础，当然，一边练习一边学习也是可以的。

### 小结
![服务器准备](https://www.hoohack.me/assets/images/2018/09/server-learning.png)
本次主要描述了为什么需要学习网络编程，同时介绍了网络交互的本质，以及计算机通信的方式，还有一些常用到的API。通过了解这些介绍，可以着手去学习基础知识，为之后编写服务器打下坚实的基础。
再次强调，网络基础真的很重要，只要开发工作涉及到网络交互，那么不管是开发还是调试功能，都会起到非常重要的作用。



原创文章，文笔有限，才疏学浅，文中若有不正之处，万望告知。

如果本文对你有帮助，请点个赞吧，谢谢^_^



