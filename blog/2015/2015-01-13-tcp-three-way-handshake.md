---
layout: post
title:  "学习TCP三次握手总结"
date:   '2015-01-13'
tags: tech
author: hoohack
categories: 计算机网络
---

## 什么是TCP三次握手

### 握手
握手是在通信电路建立之后，信息传输开始之前。 握手用于达成参数，如信息传输率，字母表，奇偶校验, 中断过程，和其他协议特性。

### TCP三次握手
TCP三次握手是TCP创建连接前的一个过程。TCP使用三次握手这个过程以保证TCP可以提供可靠的传送。



## 三次握手过程
> * 第一次：建立连接时，客户端发送syn包(SYN=1)到服务器，同时选择初始序号seq=x。此时进入SYN_SENT状态，等待服务器确认
> * 第二次：服务器接收到客户端的请求包，若同意建立连接，则想客户端发送确认包，SYN和ACK位都置1,确认号是ack=x+1，同时也为自己选择一个初始序号seq=y。此时服务器进入SYN_RCVD状态
> * 第三次：客户端接收到服务器的确认包后，还要向服务器发送确认包。确认报文段的ACK置1，确认号ack=y+1，而自己的序号seq=x+1。此包发送完毕，客户端和服务器进入ESTABLISHED状态，完成三次握手过程

## 过程图
![TCP三次握手过程图](http://7u2eqw.com1.z0.glb.clouddn.com/TCP三次握手.png)

## ACCEPT阶段
ACCEPT阶段是从已完成三次握手的队列内取走一个，与三次握手过程没有关系，即不属于TCP三次握手的任何一个阶段。

内核为任何一个给定的监听套接口维护两个队列：

     1、未完成连接队列（incomplete connection queue），每个这样的SYN分节对应其中一项：已由某个客户发出并到达服务器，而服务器正在等待完成相应的TCP三路握手过程。这些套接口处于SYN_RCVD状态。

     2、已完成连接队列（completed connection queue），每个已完成TCP三路握手过程的客户对应其中一项。这些套接口处于ESTABLISHED状态。

TCP三次握手过程发生在connect step。

ACCEPT只是把内核中的“已完成连接队列”取出完成三次握手的连接。

另一个队列是“未完成连接队列”，两个队列总和最大值是backlog(backlog参数确定了connection队列可以增长的最大长度)
