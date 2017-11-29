---
title: '[golang]golang内存管理'
date: 2017-11-16 10:04:16
tags: golang
---


最近粗略看了下golang内存初始化相关的代码，结合大牛们的一些源码分析，自己整理了一下学习总结。
#### 几个关键数据结构
- mspan
由mheap管理的页面，记录了所分配的块大小和起始地址等
- mcache
与P(可看做cpu)绑定的线程级别的本地缓存
- mcenter
全局空间的缓存，收集了各种大小(67种)的span列表
- mheap
分配内存的堆分配器，以8kb进行页管理
- fixalloc
固定尺寸的堆外对象空闲列表分配器，用来管理分配器的存储
<!--more-->

#### 内存分配逻辑
- 如果object size>32KB, 则直接使用mheap来分配空间；
- 如果object size<16Byte, 则通过mcache的tiny分配器来分配(tiny可看作是一个指针offset)；
- 如果object size在上面两者之间，首先尝试通过sizeclass对应的分配器分配;
- 如果mcache没有空闲的span， 则向mcentral申请空闲块；
- 如果mcentral也没空闲块，则向mheap申请并进行切分；
- 如果mheap也没合适的span，则向系统申请新的内存空间。

#### 内存回收逻辑
- 如果object size>32KB, 直接将span返还给mheap的自由链；
- 如果object size<32KB, 查找object对应sizeclass， 归还到mcache自由链；
- 如果mcache自由链过长或内存过大，将部分span归还到mcentral；
- 如果某个范围的mspan都已经归还到mcentral，则将这部分mspan归还到mheap页堆；
- 而mheap不会定时将内存归还到系统，但会归还虚拟地址到物理内存的映射关系，当系统需要的时候可以回收这部分内存，否则暂时先留着给Go使用。

#### 图解结构关系与内存布局
其他想说明的情况，我基本都放在图里了(画个图好累，很可能还画不准确, 窘迫...)
(图片来自我的csdn博客: <a href='http://blog.csdn.net/moxiaomomo/article/details/78546513'>blog.csdn.net/moxiaomomo<a/>)
<span style="font-szie:15px">
<img src="http://img.blog.csdn.net/20171116001110283?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbW94aWFvbW9tbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" style="max-width:1000px;">
</span>

#### 其他知识点
- 标示内存可用的常用两种方法
(1) 将所有可用内存块通过链表连接起来；
(2) 通过位图标志；
- mcentral中用到的pad字段
主要是为了避免false sharing(伪共享)的性能问题；
- 内存分配和tcmalloc理论类似
