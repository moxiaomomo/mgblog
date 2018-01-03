---
title: '分布式转码初步方案'
date: 2017-11-29 12:16:07
tags: 视频
---

#### 背景说明
现有的转码方案是一个转码worker处理整个视频的不同清晰度的转码，如果一个视频很大，那这个视频转码将非常耗时。
因此需要改进方案，要求对大多数格式的视频可以进行切片后并行转码，以此提高一个视频的转码效率。

#### 技术预研
目前搜到的参考资料，基本都是针对某几个特定格式的分布式转码方案。
当前视频转码基本都依赖于ffmpeg， 目前存在一个问题：
暂时没有找到一个合适的方法去无缝切割多种编码格式的视频，然后分别对切割出来的分片进行转码； 这个过程部分格式正常部分异常(请原谅我是非科班出身)。
计划先用low一点的方案实现，ffmpeg可以在一个完整的视频上，指定视频区间来进行转码，暂时测试了大多数格式的效果都OK；
那就每个worker的视频源都是同一个视频文件，然后并行对不同区间的视频段进行转码，所以就有了下面这个初步方案。

<!--more-->

#### 架构方案
![video_convert.png](http://123.207.231.132/img/video_convert.png)

#### 几点说明
1. 利用ffmpeg、hadoop、RBMQ， ffmpeg作为分布式存储，ffmpeg作为转码工具，RBMQ作为任务队列， 来构建一个分布式转码服务；
2. 关于方案中的几个问题：
- 每个转码worker都需要操作整个视频， 中间涉及到的上传、下载开销比较大；
- 多个过程都依赖于rbmq来完成异步转码，虽然解耦，但是会增加失败重转逻辑的复杂度；
- 如何均衡分配worker是一个关键点，比如有些视频转码的优先级比较高，有些视频比较大可能又需要更多的worker去并行处理转码。

#### 最后
架构需要改进，但应该可以在编码实现和测试的过程中逐步改进；如果最后解决了大多数格式视频都可以先切割成单独视频分片后，再针对分片来进行转码及合并的问题，那么就可以减少更多传输和存储开销，提升性能。