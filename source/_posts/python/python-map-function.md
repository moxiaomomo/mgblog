---
title: '[python]map function'
date: 2016-04-05 23:19:29
tags: python
---

### 1. 内建方法map
内建map方法可以通过一个序列的方式来实现函数之间的映射, 并且串行执行。如:
```python
import time
from datetime import datetime

def add(x, y):
    print datetime.now(), "enter add func..."
    time.sleep(2)
    print datetime.now(), "leave add func..."
    return x+y

map(add, [1,2,3], [4,5,6])
```
运行效果:
```bash
2016-04-05 15:01:49.382314 enter add func...
2016-04-05 15:01:51.383387 leave add func...
2016-04-05 15:01:51.383471 enter add func...
2016-04-05 15:01:53.385584 leave add func...
2016-04-05 15:01:53.385676 enter add func...
2016-04-05 15:01:55.387388 leave add func...
[5, 7, 9]
```
由上可见, 调用map, 相当于顺序调用了add(1,4), add(2,5)， add(3,6)方法; 一行代码实现了方法的迭代调用, 简单快捷。<br>那如果再优化一下，实现并行调用add方法， 应该怎么做呢？在python里也好实现, 利用multiprocessing模块就可以。

<!--more-->
### 2. multiprocessing模块与map方法
```python
import time
from datetime import datetime
from multiprocessing.dummy import Pool as ThreadPool
from functools import partial


def add(x, y):
    print datetime.now(), "enter add func..."
    time.sleep(2)
    print datetime.now(), "leave add func..."
    return x+y


def add_wrap(args):
    return add(*args)


if __name__ == "__main__":
    pool = ThreadPool(4) # 池的大小为4
    print pool.map(add_wrap, [(1,2),(3,4),(5,6)])
    #close the pool and wait for the work to finish
    pool.close()
    pool.join()
```
运行效果:
```bash
2016-04-05 15:10:23.690059 enter add func...
2016-04-05 15:10:23.690406 enter add func...
2016-04-05 15:10:23.690906 enter add func...
2016-04-05 15:10:25.693250 leave add func...
2016-04-05 15:10:25.693409 leave add func...
2016-04-05 15:10:25.693458 leave add func...
[3, 7, 11]
```
由上可以见， 我们已经实现了并行执行add方法

### 3. 关于multiprocessing中pool的大小与性能
```python
from multiprocessing import Pool
from multiprocessing.dummy import Pool as ThreadPool
```
以上Pool和ThreadPool模块, 一个基于进程工作, 一个基于线程工作。
<br>一般来说, 使用进程池(multiprocessing pool)来执行CPU密集型的任务, 这样可以利用到多核的好处， 理论上(池越大)核越多速度越快; <br>使用线程池(threading)来处理IO型任务, 则有个最佳线程池大小, 要根据实际情况来调节这个池的size(线程过多时, 切换线程的开销将严重影响性能)。
