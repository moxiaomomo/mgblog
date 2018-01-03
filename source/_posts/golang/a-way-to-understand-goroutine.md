---
title: '[Golang]另一角度理解goroutine'
date: 2017-11-10 10:01:11
tags: golang
---


偶然看到一条关于goroutine有趣的QA：
https://news.ycombinator.com/item?id=12459841
发现可以以另一种方式来理解goroutine，欢迎拍砖。

关键概念说明:
```
M: machine， M对应于内核线程;
P: processor, P是一种在M上运行的context, 维护了goroutine的列表;
G: goroutine核心结构, 维护了goroutine需要的栈、程序计数器以及所在的M等信息;
sysmon: GO程序启动时创建的一个用于监控管理的线程。
```
<!--more-->

#### 假设一开始没有M和P, 没有sysmon, 同时GOMAXPROCS=1

这个时候go程序以单线程模式运行，
每次执行一个G, 无法做到抢占，而必须由程序自己yield来让出CPU，
让出CPU时当前G需要将相关上下文信息保存到自己的结构中，
然后程序查找下一个等待的G并执行。

遇到systemcall时, 当前线程会新开一个线程用以接管执行之后的G，
而当前线程会阻塞知道系统调用返回， 
之后将返回结果保存到某个地方，
最后该线程退出。

#### 加入M，减少线程创建/销毁消耗，GOMAXPROCS=1不变

上面的主要问题是单线程执行任务，
遇到syscall便会在创建/销毁线程操作中消耗很多资源，
所以这时我们想复用线程。

这里我们加入了一定数量的M，每个M对应一个内核线程，
由于GOMAXPROCS=1，同时在跑的M只有一个，其他的在sleep。
与上面有所改进的是，
如果当前M遇到syscall，它不用新创建一个线程，
而是唤醒M列表中的某个M来接管执行之后的G就可以了，
而自己负责等待系统调用，
系统调用返回后将结果写到某个地方便sleep及等待唤醒。

#### 加入调度锁，支持多线程并发，GOMAXPROCS>1

上面的场景还是同时只有一个线程在运行，
我们想利用多核同时跑几个线程，
那应该怎么做呢？

我们加入一个叫调度锁的东西，
用于解决多线程从G列表争抢资源G发生冲突的场景，
设置GOMAXPROCS>1, 这时可以最多同时跑GOMAXPROCS个线程。
调度锁是一个全局锁，如果资源上锁了，那所有线程都必须等待。
这时M数量>=GOMAXPROCS, 同时在跑的M数量==GOMAXPROCS。

#### 加入P，提高并发性能

上面使用了调度锁，虽然解决了多线程并发问题，
但是由于是对列表G全局加锁，并发性能并不好。

这时我们加入了P，将结构关系G:M变为了G:P:M。
每一个M对应一个P， 而每个P维护了一个G列表，
这时每新建一个G，会按某种顺序加到某个P的G队列末尾。
这个时候不再用全局锁了，M也不用每次到全局G队列中争抢G，
只需要从P的G队列中拿出一个就绪的G运行即可。

同样如果一个M进入syscall，
它会释放P以允许其他M来接管自己的P及P所维护的G队列，
自己等待系统调用返回并保存结果后，
进入M的空闲队列并等待唤醒。

#### 加入sysmon，解决阻塞syscall问题

像上面提到的，当一个M进入syscall而释放P时具体怎么实现呢？

这里引入了sysmon这个在Go程序启动时就会创建的一个独立线程，
它用于任务的监控和管理， 内部就是一个无限循环，
它发现如果有M处于syscall状态时便将它的P及对P上的G队列抢过来，
放到其他可用的M上运行(有可能需要新建M， 也可能是唤醒某个sleep的M)。

#### 负载均衡问题

如果有些M很快把手头上的G消耗完，而有些挤压了很多，
那就会出现负载不均衡，有些任务响应太迟的问题。

所以goroutine有了这么一个调度机制(stealing work)，
当当前M发现所绑定的P上的G没有了，
它会去尝试从全局runqueue中获取一部分G任务，
如果全局runqueue也没有，那么就去其他M上'抢夺'一些G过来执行，
每次大概是抢对方队列G的一半数量。
如果最后发现没有哪个M可以'抢夺'的G任务，
那就将自己加到M空闲队列并进入sleep状态。

------------------------
至此，很粗略的概括了goroutine的协作式调度原理。