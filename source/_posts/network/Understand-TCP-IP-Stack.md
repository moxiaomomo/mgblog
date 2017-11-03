---
title: '理解TCP/IP协议栈(1)'
date: 2017-10-26 17:09:45
tags: networking
---

翻译自: https://www.cubrid.org/blog/understanding-tcp-ip-network-stack

目前我们的internet服务都是基于TCP/IP来开发的, 无法想像没有TCP/IP的Internet会是什么样子. 因此无论是在逻辑调整, 故障排除，还是发现新技术方面, 理解网络中数据的传输原理会帮助我们多方面提高服务性能表现.
本文会介绍基于Linux系统及硬件层数据流和控制流的网络协议栈体系.
TCP/IP重点特征
我应该怎么设计网络协议来保证数据的快速传输， 并保证数据有序不丢失？
TCP and IP
严格来说由于TCP和IP在不同层次上, 分开来描述它们也是可以的. 不过这里将他们描述为一个整体.
1. 面向连接
首先,  两个终端(本地和远程)建立了连接, 数据开始传输. 此处"TCP connection identifier"是两个终端的地址组合, 类型为`<local IP address, local port number, remote IP address, remote port number> `.
2. 双向字节流
双向的数据以字节流来进行通讯.
3. 按序传输
接收者按发送者发送顺序来接收数据. 因此数据是要求有序的, 一个32位int类型的数据用于标志次序.
<!--more-->
4. ACK保证可靠性
当发送者向接收者发送数据后并没有收到ACK, 发送者会重发TCP数据. 因此, 发送者会缓存没有从接收到ACK的数据.
5. 流控制
发送者会尽可能多的发送数据. 接收者告知发送者它能接收的最大字节数(空闲的buffer size, 接收窗口); 发送者基于这个最大字节数尽可能快的发送数据.
6. 拥塞控制
拥塞窗口 用于分割接收窗口,通过限制网络数据流来避免网络拥塞. 类似接收窗口, 发送者会根据接收者的拥塞窗口大小限制而使用多种策略来发送尽可能多的数据, 策略比如有TCP Vegas, Westwood, BIC, 和CUBIC. 和流控制不同的是, 拥塞控制只在发送者上实现.
数据传输
网络栈包含多层, Figure 1展示了每层的类型.
![tcp/ip layer](/img/tcp_ip_stat.png)
Figure 1: TCP/IO网络栈在数据传输时各层的处理过程.
可以简单分为三层结构:

- 用户层面

- 内核层面

- 设备层面

用户层面和内核层面的任务由CPU来处理. 用户层和内核层称为"host", 以此与设备层区分. 这里的设备指的是用于发送/接收数据包的Network Interface Card (NIC). 更准确的术语叫"LAN card".
用户层面: 首先应用程序创建要发送的数据 (Figure 1中"User data")并调用系统函数write()来发送数据. 假设socket已创建好(Figure 1中的fd). 当进入系统调用时, 系统进入内核态.
POSIX系列操作系统(包括Linux, Unix)通过文件描述符将socket暴露给应用程序. 在此类系统中, socket可看作是一种文件. 文件层执行简单校验后调用socket函数来连接到文件结构.
内核socket有两个buffer:

- send socket buffer: 用于发送数据;
- receive socket buffer: 用于接收数据.

当系统函数write被调用时, 数据将从用户区拷贝到内核区内存并且添加到socket buffer的末端, 这样按顺序发送数据. 如Figure 1, 浅灰色框包含的部分表示的是在发送buffer中的数据. 然后调用TCP.
TCP Control Block (TCB)结构用于连接socket. TCB包含请求处理TCP连接的数据, 包括: 连接状态(LISTEN, ESTABLISHED, TIME_WAIT), 接收窗口, 拥塞窗口, 序号, 重发时钟, 等等.
如果当前TCP状态允许数据传输, 一个新的TCP段(也叫packet)将被创建. 如果由于流控制这些原因无法传输数据, 系统调用会结束并且系统返回到 用户态(也就是将控制权交回应用程序).
Figure 2展示了两个TCP段:

- TCP头;

- 内容载体payload.
![tcp segment](/img/tcp_payload.png)
Figure 2: TCP Frame Structure (source).

payload包含了未接收到ACK的socket buffer内容. payload的最大长度为接收窗口大小, 拥塞窗口, maximum segment size (MSS)三者中的最大值.
然后计算TCP检验码: 计算元素包含头部信息(IP addresses, segment length, and protocol number). 然后根据TCP状态发送一个或多个packet.
实际上目前网络栈的TCP校验码由NIC计算而非内核. 不过我们认为内核计算TCP校验码更加方便.
TCP段传输到IP层: IP层在TCP段头部加上IP头信息用于IP路由. IP routing是不断跳到中间IP并最终到达目标IP的过程.
IP包传输到链路层: 链路层会使用ARP协议来找到下一个跳转IP的MAC地址. 它会将Ethernet头信息加到packet上, 这样一个host数据packet便完整了.
经过IP routing后, 经过NIC将数据传输到下一个跳转IP, 此过程会调用NIC驱动.
这个时候, 如果运行了tcpdump或Wireshark等抓包程序, 内核将会将数据包拷贝到这种程序的缓冲区. 这种情况下接收的数据包将在驱动层直接被抓取. 一般流量整形功能都是在链路层实现的.
链路驱动根据NIC制造商提供的NIC驱动通讯协议来请求传输数据包.
接收到包传输请求后, NIC将数据包从主存中拷贝到它的内存并发送到网络中. 这时根据以太网标准, 它将IFG (Inter-Frame Gap), preamble, 及CRC信息到packet中. IFG和preamble用来区分包的开始位置(专业术语, 帧), CRC用来保护数据(和TCP及IP checksum目的一样). 数据包基于以太网物理速率及流控制条件来开始传输.
当NIC发送一个包, NIC在主机CPU产生一个中断. 每个中断有他的中断标志号, 然后OS根据中断号来调用对应的驱动来处理中断. 驱动在启动时会向系统注册一个函数, 用于处理中断指令. The OS calls the interrupt handler and then the interrupt handler returns the transmitted packet to the OS(??).
目前我们讨论了当应用程序请求write时, 内核和设备处理数据传输的过程. 不过, 没有来自应用程序的write请求, 内核也可以直接通过TCP来传输数据包. 比如当接收到一个ACK且接收窗口已扩展时, 内核将创建一个包含socket buffer剩余数据的TCP段, 并发送给接收者.
