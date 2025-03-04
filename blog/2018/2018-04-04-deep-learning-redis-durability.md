---
layout: post
title: "［Redis源码阅读］redis持久化"
date: '2018-04-04'
tags: tech
author: hoohack
categories: Redis
excerpt: 'redis,c,源码分析,源码学习,redis源码,redis对象,redis对象源码,redis 4.0源码'
keywords: 'redis,c,源码分析,源码学习,redis源码,redis对象,redis对象源码,redis 4.0源码'
---

作为web开发的一员，相信大家的面试经历里少不了会遇到这个问题：redis是怎么做持久化的？

不急着给出答案，先停下来思考一下，然后再看看下面的介绍。希望看了这边文章后，你能够回答这个问题。

## 为什么需要持久化？

由于Redis是一种内存型数据库，即服务器在运行时，系统为其分配了一部分内存存储数据，一旦服务器挂了，或者突然宕机了，那么数据库里面的数据将会丢失，为了使服务器即使突然关机也能保存数据，必须通过持久化的方式将数据从内存保存到磁盘中。

对于进行持久化的程序来说，数据从程序写到计算机的磁盘的流程如下：

1、客户端发送一个写指令给数据库（此时数据在客户端的内存）



2、数据库接收到写的指令以及数据（数据此时在服务端的内存）

3、数据库发起一个系统调用，把数据写到磁盘（此时数据在内核的内存）

4、操作系统把数据传输到磁盘控制器（数据此时在磁盘缓存中）

5、磁盘控制器执行真正写入数据到物理媒介的操作（如磁盘）

如果只是考虑数据库层面，数据在第三阶段之后就安全了，在这个时候，系统调用已经发起了，即使数据库进程奔溃了，系统调用会继续进行，也能顺利将数据写入到磁盘中。
在这一步之后，在第4步内核会将数据从内核缓存保存到磁盘缓存中，但为了系统的效率问题，默认情况下不会太频繁地执行这个动作，大概会在30s执行一次，这就意味着如果这一步失败了或者就在进行这一步的时候服务器突然关机了，那么就可能会有30s的数据丢失了，这种比较普通的灾难性问题也是需要考虑的。

POSIX API也提供了一个系统调用让内核强制将缓存数据写入到磁盘中，比较常见的就是fsync系统调用。

> int fsync(int fd);

fsync函数只对由文件描述符fd指定的一个文件起作用，并且等待写磁盘操作结束后才返回。每次调用fsync时，会初始化一个写操作，然后把缓冲区的数据写入到磁盘中。fsync()函数在完成写操作的时候会阻塞进程，如果其他线程也在写同一个文件，它也会阻塞其他线程，直到完成写操作。

## 持久化

持久化是将程序数据在持久状态和瞬时状态间转换的机制。对于程序来说，程序运行中数据是在内存的，如果没有及时同步写入到磁盘，那么一旦断电或者程序突然奔溃，数据就会丢失了，只有把数据及时同步到磁盘，数据才能永久保存，不会因为宕机影像数据的有效性。而持久化就是将数据从程序同步到磁盘的一个动作过程。

![持久化](https://www.hoohack.me/assets/images/2018/04/Persistence.png)

## Redis的持久化
redis有RDB和AOF两种持久化方式。RDB是快照文件的方式，redis通过执行SAVE/BGSAVE命令，执行数据的备份，将redis当前的数据保存到`*.rdb`文件中，文件保存了所有的数据集合。AOF是服务器通过读取配置，在指定的时间里，追加redis写操作的命令到`*.aof`文件中，是一种增量的持久化方式。


### RDB
RDB文件通过SAVE或BGSAVE命令实现。
SAVE命令会阻塞Redis服务进程，直到RDB文件创建完成为止。
BGSAVE命令通过fork子进程，有子进程来进行创建RDB文件，父进程和子进程共享数据段，父进程继续提供读写服务，子进程实现备份功能。BGSAVE阶段只有在需要修改共享数据段的时候才进行拷贝，也就是COW（Copy On Write）。SAVE创建RDB文件可以通过设置多个保存条件，只要其中一个条件满足，就可以在后台执行SAVE操作。

SAVE和BGSAVE命令的实现代码如下：
    
    void saveCommand(client *c) {
        // BGSAVE执行时不能执行SAVE
        if (server.rdb_child_pid != -1) {
            addReplyError(c,"Background save already in progress");
            return;
        }
        rdbSaveInfo rsi, *rsiptr;
        rsiptr = rdbPopulateSaveInfo(&rsi);
        // 调用rdbSave函数执行备份（阻塞当前客户端）
        if (rdbSave(server.rdb_filename,rsiptr) == C_OK) {
            addReply(c,shared.ok);
        } else {
            addReply(c,shared.err);
        }
    }

    /*
    * BGSAVE 命令实现 [可选参数"schedule"]
    */
    void bgsaveCommand(client *c) {
        int schedule = 0;

        /* 当AOF正在执行时，SCHEDULE参数修改BGSAVE的效果
        * BGSAVE会在之后执行，而不是报错
        * 可以理解为：BGSAVE被提上日程
        */
        if (c->argc > 1) {
            // 参数只能是"schedule"
            if (c->argc == 2 && !strcasecmp(c->argv[1]->ptr,"schedule")) {
                schedule = 1;
            } else {
                addReply(c,shared.syntaxerr);
                return;
            }
        }

        // BGSAVE正在执行，不操作
        if (server.rdb_child_pid != -1) {
            addReplyError(c,"Background save already in progress");
        } else if (server.aof_child_pid != -1) {
            // aof正在执行，如果schedule==1，BGSAVE被提上日程
            if (schedule) {
                server.rdb_bgsave_scheduled = 1;
                addReplyStatus(c,"Background saving scheduled");
            } else {
                addReplyError(c,
                "An AOF log rewriting in progress: can't BGSAVE right now. "
                "Use BGSAVE SCHEDULE in order to schedule a BGSAVE whenever "
                "possible.");
            }
        } else if (rdbSaveBackground(server.rdb_filename,NULL) == C_OK) {// 否则调用rdbSaveBackground执行备份操作
            addReplyStatus(c,"Background saving started");
        } else {
            addReply(c,shared.err);
        }
    }


有了RDB文件之后，如果服务器关机了，或者需要新增一个服务器，重新启动数据库服务器之后，就可以通过载入RDB文件恢复之前备份的数据。
但是bgsave会耗费较长时间，不够实时，会导致在停机的时候丢失大量数据。

### AOF（Append Only File）
RDB文件保存的是数据库的键值对数据，AOF保存的是数据库执行的写命令。

AOF的实现流程有三步：

> append->write->fsync

append追加命令到AOF缓冲区，write将缓冲区的内容写入到程序缓冲区，fsync将程序缓冲区的内容写入到文件。
当AOF持久化功能处于开启状态时，服务器每执行完一个命令，就会将命令以协议格式追加写入到redisServer结构体的aof_buf缓冲区，具体的协议这里不展开阐述。

AOF的持久化发生时期有个配置选项：appendfsync。该选项有三个值：
always：所有内容写入并同步到aof文件
everysec：将aof_buf缓冲区的内容写入到AOF文件，如果距离上次同步AOF文件的
no：将aof_buf缓冲区中的所有内容写入到AOF文件，但并不对AOF文件进行同步，由操作系统决定何时进行同步，一般是默认情况下的30s。

AOF持久化模式每个写命令都会追加到AOF文件，随着服务器不断运行，AOF文件会越来越大，为了避免AOF产生的文件太大，服务器会对AOF文件进行重写，将操作相同key的相同命令合并，从而减少文件的大小。

举个例子，要保存一个员工的名字、性别等信息：
    
    > hset employee_12345 name "hoohack"
    > hset employee_12345 good_at "php"
    > hset employee_12345 gender "male"

只是录入这个哈希键的状态，AOF文件就需要保存三条命令，如果还有其他，比如删除，或者更新值的操作，那命令将会更多，文件会更大，有了重写后，就可以适当地减少文件的大小。

AOF重写的实现原理是先服务器中的数据库，然后遍历数据库，找出每个数据库中的所有键对象，获取键值对的键和值，根据键的类型对键值对进行重写。比如上面的例子，可以合并为下面的一条命令：
    
    > hset employee_12345 name "hoohack" good_at "php" gender "male"。

AOF的重写会执行大量的写入操作，Redis是单线程的，所以如果有服务器直接调用重写，服务器就不能处理其他命令了，因此Redis服务器新起了单独一个进程来执行AOF重写。

Redis执行重写的流程：
![redis rewrite](https://www.hoohack.me/assets/images/2018/04/redis-rewrite.png)

在子进程执行AOF重写时，服务端接收到客户端的命令之后，先执行客户端发来的命令，然后将执行后的写命令追加到AOF缓冲区中，同时将执行后的写命令追加到AOF重写缓冲区中。
等到子进程完成了重写工作后，会发一个完成的信号给服务器，服务器就将AOF重写缓冲区中的所有内容追加到AOF文件中，然后原子性地覆盖现有的AOF文件。

### RDB和AOF的优缺点
RDB持久化方式可以只通过服务器读取数据就能加载备份中的文件到程序中，而AOF方式必须创建一个伪客户端才能执行。

RDB的文件较小，保存了某个时间点之前的数据，适合做灾备和主从同步。

RDB备份耗时较长，如果数据量大，在遇到宕机的情况下，可能会丢失部分数据。另外，RDB是通过配置使达到某种条件的时候才执行，如果在这段时间内宕机，那么这部分数据也会丢失。

AOF方式，在相同数据集的情况下，文件大小会比RDB方式的大。

AOF的持久化方式也是通过配置的不同，默认配置的是每秒同步，最快的模式是同步每一个命令，最坏的方式是等待系统执行fsync将缓冲同步到磁盘文件中，大部分操作系统是30s。通常情况下会配置为每秒同步一次，所以最多会有1s的数据丢失。

### 怎样的同步方式更好？
RDB和AOF方式结合。起一个定时任务，每小时备份一份服务器当前状态的数据，以日期和小时命名，另外起一个定时任务，定时删除无效的备份文件（比如48小时之前）。AOF配置为1s一次。这样一来，最多会丢失1s的数据，同时如果redis发生雪崩，也能迅速恢复为前一天的状态，不至于停止服务。

## 总结
Redis的持久化方案也不是一成不变的，纸上的理论还需要结合实践成果来证明其可行性。

原创文章，文笔有限，才疏学浅，文中若有不正之处，万望告知。





参考文章：
[http://oldblog.antirez.com/post/redis-persistence-demystified.html](http://oldblog.antirez.com/post/redis-persistence-demystified.html)

[http://blog.httrack.com/blog/2013/11/15/everything-you-always-wanted-to-know-about-fsync/](http://blog.httrack.com/blog/2013/11/15/everything-you-always-wanted-to-know-about-fsync/)