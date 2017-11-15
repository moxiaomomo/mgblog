---
title: '理解TCP/IP协议栈(3)'
date: 2017-11-07 14:14:27
tags: networking
---

翻译自: https://www.cubrid.org/blog/understanding-tcp-ip-network-stack

### 如何处理中断及接收包
中断处理很复杂, 而你需要理解与之相关的到达包处理的性能. Figure 5展示了一个中断处理的流程.
![irq_handle.png](http://www.moguang.me/img/irq_handle.png)
Figure 5: Processing Interrupt, softirq, and Received Packet.
<!--more-->
- 假设CPU 0正在执行一个用户程序. 这时NIC接收到一个数据包并在CPU 0上产生一个中断. 接着CPU执行内核中断处理handler. 该handler根据中断号来调对应的驱动中断handler. 驱动释放已传输的包后调用napi_schedule()函数来处理到达的包. 该函数会请求softirq(软中断).
- 在驱动中断handler执行结束后, 控制权交还给内核handler. 内核handler便接着执行该softirq的处理.
- 在中断上下文执行后, softirq上下文会被执行. 中断上下文和softirq上下文在一个独立的线程中执行. 然而他们用的是不同栈, 而且中断上下文会阻塞硬件中断; 不过softirq上下文不会阻塞硬件中断.softirq handler中处理到达包的函数是net_rx_action() function. 该函数会调用驱动函数poll(), 而poll()函数会调用netif_receive_skb()函数并将到达包一个个发送到上层程序. 处理softirq后, 应用会回到上次进行系统调用时的中止位置并重新执行.
- 此时接收到中断信号的CPU已经完整的处理了一遍到达的数据包. 像Linux, BSD, 和Microsoft Windows这些系统, 这些处理流程基本是一致的.
- 当你检查CPU利用率时, 有时你会发现只有一个CPU在执行softirq. 这个情况是目前介绍到的包处理流程方式引起的. 为了解决这个问题, 多队列NIC, RSS, 和RPS等技术被开发了出来.
### 数据结构
下面是一些重要的数据结构, 现在来看看相关代码.
``sk_buff structure``
首先是表示一个包结构的sk_buff或skb. Figure 6 展示了 sk_buff 的部分结构. 实际上这些函数在不断完善, 要比图示复杂的多. 不过基础的功能还是基本一样的.
![irq_handle.png](http://www.moguang.me/img/skbuff.png)
Figure 6: Packet Structure sk_buff.

#### 包含包数据及元数据
该结构直接包含了包数据或通过指针指向数据. Figure 6, 一些包(从Ethernet到buffer)包含了数据指针而其他数据(flags)存在于内存页中.
一些必须的信息，比如header以及payload大小等保存在元数据区域. 比如Figure 6中的mac_header, network_header, transport_header有对应的指针分别指向Ethernet header, IP header及TCP header的开始位置. 这样使得TCP处理起来更简易.
#### 如何增加/删除一个header
在网络栈中的不同层，header会根据实际情况增加删除. 指针使得流程处理更有效率. 比如要删除Ethernet header, just increase the head pointer？
#### 如何合并包/拆包
链表使得在sokcet buffer/packt chain中增加/删除包的payload内容这些task更有效率. 指针next, prev就是用来做这些操作的.
#### 快速分配和释放
由于在创建包时会初始化一个数据结构体, 会用到quick allocator.比如如果数据在10Gb的以太网上传输, 每秒将有超过一百万次的包创建/回收操作.
#### TCP控制块
其次专门有一个结构表示TCP连接, 之前笼统的称为TCP控制块. Linux中使用tcp_sock来表示这个结构. Figure 7中, 你可以看到文件与socket, tcp_socket之间的关系.
![irq_handle.png](http://www.moguang.me/img/tcp_conn_struct.png)
Figure 7: TCP Connection Structure.

- 当发生系统调用时, 内核会去搜索应用所传递过来的文件描述符. 在Unix系列系统中, socket, file和文件系统存储设备被抽象成file的概念. 因此file结构包含了尽可能少的信息. 对于socket, 一个socket结构包含了socket相关的信息以及作为指针指向socket, socket同时又指向tcp_sock. tcp_sock 可分类成sock, inet_sock等, 支持除TCP外的多种协议. 可以看作是一种多态polymorphism.
- TCP协议所用到的状态信息保存tcp_sock. 比如序列号, 接收窗口, 拥塞控制, 以及重传时钟等都保存在tcp_sock.
- 用于发送/接收的socket buffer为sk_buff列表，他们包含了tcp_sock. IP路由结果结构dst_entry用于避免过于频繁的routing. dst_entry允许用于简单的ARP结果搜索, 也就是MAC地址. dst_entry是routing表的一部分. routing表很复杂而本文不会详细讨论它. 搜索用于传输包的NIC时会用到dst_entry. NIC信息保存在net_device结构中.
- 因此通过查找文件，可以容易找到利用指针处理TCP连接的所有数据结构(from the file to the driver). 结构的大小就是一个TCP连接在内存中占用的内存， 大概是几KB (不包括包数据). 随着用到更多功能，内存占用大小也有所增加.
- 最后看下TCP连接的lookup表. 这里用了一张hash表去查找到达包所属的TCP连接. hash值通过传入的包头<source IP, target IP, source port, target port>来通过Jenkins hash算法计算. 据说使用了防御攻击hash表的hash函数.
