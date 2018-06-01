---
title: 基于环形队列和令牌桶实现的限流模块
date: 2018-06-01 10:17:40
tags: golang
---

### 概述

在分布式服务架构下，比如微服务架构，一般需要构建一个独立的gateway模块。gateway模块的主要作用包括流量控制，规则路由，负载均衡，鉴权，熔断等等；而gateway一般是stateless的，因此可自由弹性伸缩集群规模，以应对实际的访问场景。
本文主要介绍通过golang实现限流模块的一种方案，其关键点主要有两个：

- 环形队列(ring queue)
- 令牌桶算法(token bucket)

<!--more-->

### 基于环形队列(ring queue)

```golang
type Queue struct {
	mtx      sync.RWMutex
	head     int64
	tail     int64
	capacity int64
	elemcnt  int64
	data     []interface{}
}

// 将一个元素推入队列n次
func func (q *Queue) AtomEnqueue(d interface{}, n int64) bool {
}

// 根据自定义方法的判定结果，将符合条件的元素推出队列
func (q *Queue) DequeueBy(fc func(interface{}) bool) []interface{
}
```

环形队列在限流中的作用是，提供一个状态保存空间，用以判断当前访问频率是否已达到限制值。

在使用这种队列之前，尝试过用time.Tick()时钟方法来实现令牌的定期发放，容易实现但是性能是不高的。因为据说在频率超过1000rps时， 不建议用time.Tick, 具体可以参考这里[controlling throughput with rate.limiter][2] 。

另外如果用到time.Tick()， 一般就需要新开goroutine来进行定期执行了。虽然并没有消耗太多性能，但能避免总是好的。而且如果限流是要实现到针对ip，或者user级别的粒度，那开销就很难忽略了。

而环形队列的一个好处是内存从初始化后就基本是固定的，队列里的每一个元素存放的是每个请求的状态(当前示例里存的是访问时间戳)，在请求到达时查询流量状态的同时，进行队列元素的更新(有别于time.Tick的定期性，这个是依靠请求到达来触发的，也就不需要依赖到其他系统资源)。另一个好处就是访问速度快，也没有过多的内存拷贝。其中要注意的是解决并发竞争的问题，用读写锁是其中一种解决方法。

### 基于令牌桶(token bucket)算法

```golang
type TokenBucket struct {
	mutex    sync.Mutex
	capacity int64
	interval time.Duration
	tsQ      *Queue
}

// 获取token， 成功返回true， 失败返回false及需要等待的时间
func (tb *TokenBucket) Take(n int64) (bool, time.Duration) {
}

// 获取token， 成功返回true，失败则等待直至超时
func (tb *TokenBucket) Wait(n int64, t time.Duration) bool {
}
```
常用的限流算法有两种: 漏桶算法， 令牌桶算法。网上有更详细的算法描述，这里不展开讲了。

我这里采取的是类似令牌桶的算法，原算法是每隔一段时间往桶里放一个token，有请求来时就拿走token，没有token时请求就等待（Wait）或直接返回失败（Take）；这里的实现有所区别：

- 首先是实现中并不是放token， 而是记录每个到达请求的时间戳，环形队列中有空闲的元素才能记录成功，否则等待或返回失败， 
- 另外触发时机也不一样，不是基于时钟的周期触发，而是依靠请求到达来触发

### 源码地址及测试用例

[github项目地址][1]

未严格验证，欢迎各位大大帮忙甩bug。其中测试用例参考如下：
```shell
$ go test
[1 2 3 4 5 6]
[1 2 3 4 5 6 7 8]
[3 4 5 6 7 8 9 10]
[9 10 3 4 5 6 7 8]
[10 11 12 13 14 15 16 17]
[10 11 12 13 14]
[15 16 17]
[]
[9 10 haha 15 3 6]
[9 10]
take-n:1 time-used:3.327341023s suc-cnt:4000 fail-cnt:9996000
take-n:2 time-used:3.334162956s suc-cnt:2000 fail-cnt:9998000
take-n:1 time-used:1.257582083s suc-cnt:1499 fail-cnt:1001
take-n:1 time-used:2.250956732s suc-cnt:2500 fail-cnt:0
take-n:2 time-used:1.254886958s suc-cnt:999 fail-cnt:1501
take-n:2 time-used:4.251330391s suc-cnt:2499 fail-cnt:1
take-n:2 time-used:2.251117604s suc-cnt:2500 fail-cnt:0
PASS
ok      github.com/moxiaomomo/token-bucket23.931s
```

  [1]: https://github.com/moxiaomomo/token-bucket
  [2]: https://rodaine.com/2017/05/x-files-time-rate-golang

慕课网同步链接：https://www.imooc.com/article/31788
