---
layout: post
title: "［从0到1编写服务器］TCP连接建立与断开状态变化"
date: '2018-09-27'
tags: tech
author: hoohack
categories: Network
excerpt: '网络编程,服务器编程,LinuxC,socket,tcp/ip'
keywords: '网络编程,服务器编程,LinuxC,socket,tcp/ip'
---


[上篇](https://www.hoohack.me/2018/09/05/webserver-zero-to-one-basic)介绍了socket编程的准备知识，是不是有一种很想马上就开始了解网络编程，甚至开始写点代码的感觉，别着急，网络编程中还有一个比较重要的概念是TCP/IP，中文名称叫网络传输协议，本质上，TCP/IP是一种协议，同时也是网络编程中最重要的协议之一。TCP/IP涉及到的内容实在太多，无奈笔者才疏学浅，无法把整个TCP/IP介绍给大家，这篇文章的目的主要是基于上一篇文章的前提下，介绍TCP连接三次握手和断开连接四次挥手究竟做了什么？socket的状态有哪些？在各个API执行的过程中，socket的状态是怎么变化的？希望通过这篇文章，能让大家对在TCP连接建立与断开过程中，socket的整个状态变化流程有更深入的了解。

### 几个术语

SYN : 同步序列编号，Synchronize Sequence Numbers，仅在三次握手建立TCP连接时有效。表示一个新的TCP连接请求。 

ACK : 确认编号，Acknowledgement Number，对TCP请求的确认标志，同时提示对端系统已经成功接收所有数据。 

FIN : 结束标志，FINISH，用来结束一个TCP会话，但对应端口仍处于开放状态，准备接收后续数据。 



### TCP三次握手

建立一次连接会有下面的流程

1）服务器通过socket（初始化socket）、bind（绑定ip端口）、listen（开始监听服务）完成一次socket连接的建立，并调用accept函数准备好接收外部请求连接，这一步被称为被动打开

2）客户端通过socket（初始化socket）完成了连接建立，调用connect函数发起主动打开，此时客户端会发送一个SYN分节J给服务器，J的作用是告诉服务器，在接下来的数据传输过程中，J是客户端在连接中传输数据的初始序列号

3）服务器收到客户端的SYN分节后，需要回复确认信号ACK，J+1，代表服务器已经收到客户端的请求，且已经确认了客户端的初始序列号，同时服务器会发送SYN分节K给客户端，K的作用是告诉客户端，在接下来的数据传输过程中，K是服务器在连接中传输数据的初始序列号

4）客户端收到服务器的SYN分节后，回复确认信号（ACK）K+1。连接建立成功。

整个建立连接过程至少需要三个分节，因此被称为TCP三路握手。下面是TCP三次握手的流程图：

![tcp-status](https://www.hoohack.me/assets/images/2018/09/tcp-3shake.gif)



### TCP四次握手

TCP通过三次握手建立连接，然而，断开连接需要四次握手，TCP断开连接的流程描述如下：

1、客户端应用程序调用close函数，TCP中，称首先调用close的那一端为主动关闭，主动关闭的这一端发送FIN分节，意味着已经完成发送数据了；

2、另一端（即服务器），接收到关闭请求，也收到FIN分节，开始执行被动关闭操作，称为被动关闭的一端。这个FIN分节由TCP确认，发送一个确认分节ACK给客户端，收到的FIN同时也是作为文件结束符传递给应用，意味着应用程序在接收了FIN之后就不会再接收连接上的数据；

3、之后，接收到文件结束符的应用程序会关闭它的socket，服务端的TCP也会发送一个FIN分节；

4、客户端接收到FIN分节，并确认最后的关闭操作，发送ACK分节给服务端。

整个连接过程中，每一端的关闭和确认关闭都各自需要一个FIN和ACK分节，整个过程通常共需要4个分节，因此也称为TCP四次握手。下面是TCP关闭连接四次握手的流程图：

![tcp-status](https://www.hoohack.me/assets/images/2018/09/tcp-4shake.gif)



实际上，网络上的传输是没有连接的，包括TCP也是一样，TCP所谓的"连接"与"断开连接"，其实只是一种虚拟的叫法，只不过是在通讯的双方维护一个"连接状态"，让它看上去好像有连接一样，所以，了解TCP的状态变换是非常重要的，接下来介绍socket的状态以及在TCP建立连接与断开连接过程中，socket状态的变化。

### socket状态

socket共定义了11种状态：LISTEN、SYN-SENT、SYN-RECEIVED、ESTABLISHED、FIN-WAIT-1、FIN-WAIT-1、CLOSE_WAIT、FIN_WAIT、LAST-ACK、TIME-WAIT、CLOSING、CLOSED。

我们假定A服务请求连接B服务。

LISTEN：开始建立连接，此时socket已经初始化成功，正在等待连接

SYN-SENT：A成功发送连接请求给B，等待对方响应

SYN-RECEIVED：B接收到A的连接请求，并回复了确认进行连接给A，此状态表示正在等待A也回复确认收到此消息

ESTABLISHED：表示连接已经成功建立；这个状态是连接阶段中进行数据传输的正常状态

FIN-WAIT-1：等待主动断开连接请求的确认，或者并发请求被拒绝的断开连接，这种状态通常持续时间很短，比较难捕捉

FIN-WAIT-2：等待B断开连接操作，这个状态通常持续时间也很短，但是如果B发生阻塞或者其他原因没有关闭连接，那么这个状态就会持续较长时间

CLOSE-WAIT：B已经收到了A的断开连接请求，正在等待本地应用程序发送断开连接请求

CLOSING：A正在等待B的关闭连接确认信号，当A接收到本地程序断开连接的请求后，就发送断开连接请求给B，并进入此状态

LAST-ACK：B等待断开连接的确认信号

TIME-WAIT：等待一段时间，确认B接收到A的关闭连接确认信号

CLOSED：连接完全关闭

那么在一个完整的TCP连接过程，TCP的状态是怎么转换的呢？

先来看看下面这张图，是UNIX网络编程中，TCP状态变化的经典流程图：

![TCP-status](https://www.hoohack.me/assets/images/2018/09/tcp-classic.png)

在server端，调用socket函数创建一个sockect，函数返回一个socket文件描述符，调用bind函数绑定ip地址和端口，之后调用listen函数，scokect变成正在监听的socket，进入LISTEN状态，并调用accept等待请求（想要测试LISTEN状态，可以建立socket=>bind=>listen，然后sleep10秒之后退出，启动server之后马上用netstat命令可以看到）

客户端调用connect，主动打开文件描述符，请求建立连接，此时会触发TCP的三次握手，进入SYN-SENT状态

此时服务器调用了accept正在阻塞阶段，接收到客户端的连接请求后，进入SYN-RECEIVED状态，回复确认报文给客户端，等待客户端确认连接

客户端收到服务器的连接确认报文SYN后，此时TCP已经完成了三次握手，connect函数返回，确认建立连接，转成ESTABLISHED状态，并发送确认报文ACK给服务器。（当第二次握手完成，握手步骤的第二个segment被client接收到的时候，connect函数返回。）

在服务器侧，接收到客户端的确认报文，accept也收到返回，服务器也进入ESTALISHED状态。（握手步骤的第三个分节被server接收到的时候，accept函数返回，即connect返回后经过一半的RTT时间才返回。）

客户端和服务端成功建立"连接"后，就进行开始通信，此时会调用read/write函数进行读写数据，读写数据完毕后，就准备关闭连接

假定客户端应用程序，收到了服务器的响应报文，完成了通信过程，准备关闭应用，调用close函数发起主动关闭，此时客户端进入FIN-WAIT1状态，发送FIN分节到服务器，等待服务器确认

服务端收到断开连接请求分节FIN（M），此时read函数返回0，服务器准备执行关闭操作，不再接收任何数据，并发送确认报文ACK（M+1）给客户端

客户端收到确认分节后，表示服务器已经接收到断开连接请求，此时不会再发送数据并等待服务器断开连接，进入FIN-WAIT2状态

服务器发送确认断开连接后，调用close函数，断开连接，close函数成功返回后，发送FIN分节给客户端，进入LASK-ACK状态，等待客户端的确认

客户端收到服务器的断开连接分节FIN（N）后，发送确认分节ACK给服务器，并进入TIME-WAIT状态，此时会等待足够的时间，大约是最长分节生命期的2倍（2MSL），确认服务器收到断开连接的确认分节，之后就会消失。 服务器收到客户端的断开连接的确认分节ACK（N+1）后，表示连接已经完全断开了，此时进入CLOSED状态 

看完上面的一大段文字可能有点枯燥，甚至有点懵，来看看这张动画图：

![tcp-status](https://www.hoohack.me/assets/images/2018/09/tcp-status.gif)

### 总结

本次主要介绍了TCP状态，以及在TCP连接建立和断开过程中，TCP状态的变化，掌握了这个，对理解网络编程中，各个流程的状态有比较大的帮助，比如在排查服务是否启动的时候，就可以通过`netstat -nlp | grep '端口号'`，查看服务的状态描述字符串，如果是LISTEN状态表示服务已经正常启动，如果服务处于其他状态，则可以通过服务状态来进一步排查问题。

原创文章，文笔有限，才疏学浅，文中若有不正之处，万望告知。

如果本文对你有帮助，请点个赞吧，谢谢^_^



