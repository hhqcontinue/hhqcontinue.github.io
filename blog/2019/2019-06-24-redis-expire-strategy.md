---
layout: post
title: "理解Redis的内存回收机制"
date: '2019-06-24'
tags: tech
author: hoohack
categories: Redis
excerpt: 'Redis,Redis回收机制,Redis内存回收,Redis惰性删除,Redis面试题,Redis源码'
keywords: 'Redis,Redis回收机制,Redis内存回收,Redis惰性删除,Redis面试题,Redis源码'
---

之前看到过一道面试题：Redis的过期策略都有哪些？内存淘汰机制都有哪些？手写一下LRU代码实现？笔者结合在工作上遇到的问题学习分析，希望看完这篇文章能对大家有所帮助。

## 从一次不可描述的故障说起
问题描述：一个依赖于定时器任务的生成的接口列表数据，时而有，时而没有。

### 怀疑是Redis过期删除策略
排查过程长，因为手动执行定时器，set数据没有报错，但是set数据之后不生效。

set没报错，但是set完再查的情况下没数据，开始怀疑Redis的过期删除策略（准确来说应该是Redis的内存回收机制中的数据淘汰策略触发内存上限淘汰数据。），导致新加入Redis的数据都被丢弃了。最终发现故障的原因是因为配置错了，导致数据写错地方，并不是Redis的内存回收机制引起。



通过这次故障后思考总结，如果下一次遇到类似的问题，在怀疑Redis的内存回收之后，如何有效地证明它的正确性？如何快速证明猜测的正确与否？以及什么情况下怀疑内存回收才是合理的呢？下一次如果再次遇到类似问题，就能够更快更准地定位问题的原因。另外，Redis的内存回收机制原理也需要掌握，明白是什么，为什么。

花了点时间查阅资料研究Redis的内存回收机制，并阅读了内存回收的实现代码，通过代码结合理论，给大家分享一下Redis的内存回收机制。

## 为什么需要内存回收？
* 1、在Redis中，set指令可以指定key的过期时间，当过期时间到达以后，key就失效了；

* 2、Redis是基于内存操作的，所有的数据都是保存在内存中，一台机器的内存是有限且很宝贵的。

基于以上两点，为了保证Redis能继续提供可靠的服务，Redis需要一种机制清理掉不常用的、无效的、多余的数据，失效后的数据需要及时清理，这就需要内存回收了。

## Redis的内存回收机制
Redis的内存回收主要分为过期删除策略和内存淘汰策略两部分。

### 过期删除策略
删除达到过期时间的key。

#### 1、定时删除
对于每一个设置了过期时间的key都会创建一个定时器，一旦到达过期时间就立即删除。该策略可以立即清除过期的数据，对内存较友好，但是缺点是占用了大量的CPU资源去处理过期的数据，会影响Redis的吞吐量和响应时间。

#### 2、惰性删除
当访问一个key时，才判断该key是否过期，过期则删除。该策略能最大限度地节省CPU资源，但是对内存却十分不友好。有一种极端的情况是可能出现大量的过期key没有被再次访问，因此不会被清除，导致占用了大量的内存。

> 在计算机科学中，懒惰删除（英文：lazy deletion）指的是从一个散列表（也称哈希表）中删除元素的一种方法。在这个方法中，删除仅仅是指标记一个元素被删除，而不是整个清除它。被删除的位点在插入时被当作空元素，在搜索之时被当作已占据。

#### 3、定期删除
每隔一段时间，扫描Redis中过期key字典，并清除部分过期的key。该策略是前两者的一个折中方案，还可以通过调整定时扫描的时间间隔和每次扫描的限定耗时，在不同情况下使得CPU和内存资源达到最优的平衡效果。

**在Redis中，同时使用了定期删除和惰性删除。** 

### 过期删除策略原理

为了大家听起来不会觉得疑惑，在正式介绍过期删除策略原理之前，先给大家介绍一点可能会用到的相关Redis基础知识。

#### redisDb结构体定义
我们知道，Redis是一个键值对数据库，对于每一个redis数据库，redis使用一个redisDb的结构体来保存，它的结构如下：

```c
    typedef struct redisDb {
            dict *dict;                 /* 数据库的键空间，保存数据库中的所有键值对 */
            dict *expires;              /* 保存所有过期的键 */
            dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
            dict *ready_keys;           /* Blocked keys that received a PUSH */
            dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
            int id;                     /* 数据库ID字段，代表不同的数据库 */
            long long avg_ttl;          /* Average TTL, just for stats */
    } redisDb;
```

从结构定义中我们可以发现，对于每一个Redis数据库，都会使用一个字典的数据结构来保存每一个键值对，dict的结构图如下：

![dict struct](https://www.hoohack.me/assets/images/2019/06/redis_dict_struct.png)

以上就是过期策略实现时用到比较核心的数据结构。程序=数据结构+算法，介绍完数据结构以后，接下来继续看看处理的算法是怎样的。

#### expires属性
redisDb定义的第二个属性是expires，它的类型也是字典，Redis会把所有过期的键值对加入到expires，之后再通过定期删除来清理expires里面的值。加入expires的场景有：

1、set指定过期时间expire

如果设置key的时候指定了过期时间，Redis会将这个key直接加入到expires字典中，并将超时时间设置到该字典元素。

2、调用expire命令

显式指定某个key的过期时间

3、恢复或修改数据

从Redis持久化文件中恢复文件或者修改key，如果数据中的key已经设置了过期时间，就将这个key加入到expires字典中

以上这些操作都会将过期的key保存到expires。redis会定期从expires字典中清理过期的key。

#### Redis清理过期key的时机
1、Redis在启动的时候，会注册两种事件，一种是时间事件，另一种是文件事件。（可参考[启动Redis的时候，Redis做了什么](https://www.hoohack.me/blog/2018/2018-05-26-read-redis-src-how-server-start)）时间事件主要是Redis处理后台操作的一类事件，比如客户端超时、删除过期key；文件事件是处理请求。

在时间事件中，redis注册的回调函数是serverCron，在定时任务回调函数中，通过调用databasesCron清理部分过期key。（这是定期删除的实现。）
```c  
    int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData)
    {
        …
        /* Handle background operations on Redis databases. */
        databasesCron();
        ...
    }
```

2、每次访问key的时候，都会调用expireIfNeeded函数判断key是否过期，如果是，清理key。（这是惰性删除的实现。）
```c
    robj *lookupKeyRead(redisDb *db, robj *key) {
        robj *val;
        expireIfNeeded(db,key);
        val = lookupKey(db,key);
         ...
        return val;
    }
```

3、每次事件循环执行时，主动清理部分过期key。（这也是惰性删除的实现。）
```c
    void aeMain(aeEventLoop *eventLoop) {
        eventLoop->stop = 0;
        while (!eventLoop->stop) {
            if (eventLoop->beforesleep != NULL)
                eventLoop->beforesleep(eventLoop);
            aeProcessEvents(eventLoop, AE_ALL_EVENTS);
        }
    }

    void beforeSleep(struct aeEventLoop *eventLoop) {
           ...
           /* Run a fast expire cycle (the called function will return
            - ASAP if a fast cycle is not needed). */
           if (server.active_expire_enabled && server.masterhost == NULL)
               activeExpireCycle(ACTIVE_EXPIRE_CYCLE_FAST);
           ...
       }
```

#### 过期策略的实现
我们知道，Redis是以单线程运行的，在清理key是不能占用过多的时间和CPU，需要在尽量不影响正常的服务情况下，进行过期key的清理。过期清理的算法如下：
- 1、server.hz配置了serverCron任务的执行周期，默认是10，即CPU空闲时每秒执行十次。
- 2、每次清理过期key的时间不能超过CPU时间的25%：timelimit = 1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100;
    比如，如果hz=1，一次清理的最大时间为250ms，hz=10，一次清理的最大时间为25ms。
- 3、如果是快速清理模式（在beforeSleep函数调用），则一次清理的最大时间是1ms。
- 4、依次遍历所有的DB。
- 5、从db的过期列表中随机取20个key，判断是否过期，如果过期，则清理。
- 6、如果有5个以上的key过期，则重复步骤5，否则继续处理下一个db
- 7、在清理过程中，如果达到CPU的25%时间，退出清理过程。

从实现的算法中可以看出，这只是基于概率的简单算法，且是随机的抽取，因此是无法删除所有的过期key，通过调高hz参数可以提升清理的频率，过期key可以更及时的被删除，但hz太高会增加CPU时间的消耗。

#### 删除key
Redis4.0以前，删除指令是del，del会直接释放对象的内存，大部分情况下，这个指令非常快，没有任何延迟的感觉。但是，如果删除的key是一个非常大的对象，比如一个包含了千万元素的hash，那么删除操作就会导致单线程卡顿，Redis的响应就慢了。为了解决这个问题，在Redis4.0版本引入了unlink指令，能对删除操作进行“懒”处理，将删除操作丢给后台线程，由后台线程来异步回收内存。

实际上，在判断key需要过期之后，真正删除key的过程是先广播expire事件到从库和AOF文件中，然后在根据redis的配置决定立即删除还是异步删除。

如果是立即删除，Redis会立即释放key和value占用的内存空间，否则，Redis会在另一个bio线程中释放需要延迟删除的空间。

#### 小结
总的来说，Redis的过期删除策略是在启动时注册了serverCron函数，每一个时间时钟周期，都会抽取expires字典中的部分key进行清理，从而实现定期删除。另外，Redis会在访问key时判断key是否过期，如果过期了，就删除，以及每一次Redis访问事件到来时，beforeSleep都会调用activeExpireCycle函数，在1ms时间内主动清理部分key，这是惰性删除的实现。

Redis结合了定期删除和惰性删除，基本上能很好的处理过期数据的清理，但是实际上还是有点问题的，如果过期key较多，定期删除漏掉了一部分，而且也没有及时去查，即没有走惰性删除，那么就会有大量的过期key堆积在内存中，导致redis内存耗尽，当内存耗尽之后，有新的key到来会发生什么事呢？是直接抛弃还是其他措施呢？有什么办法可以接受更多的key？

### 内存淘汰策略
Redis的内存淘汰策略，是指内存达到maxmemory极限时，使用某种算法来决定清理掉哪些数据，以保证新数据的存入。

#### Redis的内存淘汰机制
* noeviction: 当内存不足以容纳新写入数据时，新写入操作会报错。
* allkeys-lru：当内存不足以容纳新写入数据时，在键空间（``server.db[i].dict``）中，移除最近最少使用的 key（这个是最常用的）。
* allkeys-random：当内存不足以容纳新写入数据时，在键空间（``server.db[i].dict``）中，随机移除某个 key。
* volatile-lru：当内存不足以容纳新写入数据时，在设置了过期时间的键空间（``server.db[i].expires``）中，移除最近最少使用的 key。
* volatile-random：当内存不足以容纳新写入数据时，在设置了过期时间的键空间（``server.db[i].expires``）中，随机移除某个 key。
* volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间（``server.db[i].expires``）中，有更早过期时间的 key 优先移除。

> 在配置文件中，通过maxmemory-policy可以配置要使用哪一个淘汰机制。

#### 什么时候会进行淘汰？
Redis会在每一次处理命令的时候（processCommand函数调用freeMemoryIfNeeded）判断当前redis是否达到了内存的最大限制，如果达到限制，则使用对应的算法去处理需要删除的key。伪代码如下：

```c
    int processCommand(client *c)
    {
        ...
        if (server.maxmemory) {
            int retval = freeMemoryIfNeeded();  
        }
        ...
    }
```

#### LRU实现原理
在淘汰key时，Redis默认最常用的是LRU算法（Latest Recently Used）。Redis通过在每一个redisObject保存lru属性来保存key最近的访问时间，在实现LRU算法时直接读取key的lru属性。

具体实现时，Redis遍历每一个db，从每一个db中随机抽取一批样本key，默认是3个key，再从这3个key中，删除最近最少使用的key。实现伪代码如下：

```c 
    keys = getSomeKeys(dict, sample)
    key = findSmallestIdle(keys)
    remove(key)
```

3这个数字是配置文件中的maxmeory-samples字段，也是可以可以设置采样的大小，如果设置为10，那么效果会更好，不过也会耗费更多的CPU资源。

以上就是Redis内存回收机制的原理介绍，了解了上面的原理介绍后，回到一开始的问题，在怀疑Redis内存回收机制的时候能不能及时判断故障是不是因为Redis的内存回收机制导致的呢？

## 回到问题原点
如何证明故障是不是由内存回收机制引起的？

根据前面分析的内容，如果set没有报错，但是不生效，只有两种情况：

* 1、设置的过期时间过短，比如，1s？
* 2、内存超过了最大限制，且设置的是noeviction或者allkeys-random。

因此，在遇到这种情况，首先看set的时候是否加了过期时间，且过期时间是否合理，如果过期时间较短，那么应该检查一下设计是否合理。

如果过期时间没问题，那就需要查看Redis的内存使用率，查看Redis的配置文件或者在Redis中使用info命令查看Redis的状态，maxmemory属性查看最大内存值。如果是0，则没有限制，此时是通过total_system_memory限制，对比used_memory与Redis最大内存，查看内存使用率。

如果当前的内存使用率较大，那么就需要查看是否有配置最大内存，如果有且内存超了，那么就可以初步判定是内存回收机制导致key设置不成功，还需要查看内存淘汰算法是否noeviction或者allkeys-random，如果是，则可以确认是redis的内存回收机制导致。如果内存没有超，或者内存淘汰算法不是上面的两者，则还需要看看key是否已经过期，通过ttl查看key的存活时间。如果运行了程序，set没有报错，则ttl应该马上更新，否则说明set失败，如果set失败了那么就应该查看操作的程序代码是否正确了。

![trouble_process](https://www.hoohack.me/assets/images/2019/06/trouble_process.png)

## 总结
Redis对于内存的回收有两种方式，一种是过期key的回收，另一种是超过redis的最大内存后的内存释放。

对于第一种情况，Redis会在：

1、每一次访问的时候判断key的过期时间是否到达，如果到达，就删除key

2、redis启动时会创建一个定时事件，会定期清理部分过期的key，默认是每秒执行十次检查，每次过期key清理的时间不超过CPU时间的25%，即若hz=1，则一次清理时间最大为250ms，若hz=10，则一次清理时间最大为25ms。

对于第二种情况，redis会在每次处理redis命令的时候判断当前redis是否达到了内存的最大限制，如果达到限制，则使用对应的算法去处理需要删除的key。

![理解Redis内存回收机制](https://www.hoohack.me/assets/images/2019/06/understanding-redis-recall.png)

看完这篇文章后，你能回答文章开头的面试题了吗？

## 思考
留下一道思考题，我们知道，Redis是单线程的，单线程的redis还包含了这么多的任务每一次处理命令的线程都包含：处理命令、清理过期key、处理内存回收这些任务，为什么还能这么快？里面做了什么优化？后续再探索这个问题，敬请关注。

原创文章，文笔有限，才疏学浅，文中若有不正之处，万望告知。

如果本文对你有帮助，请点个赞吧，谢谢^_^




