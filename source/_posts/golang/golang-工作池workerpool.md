---
title: '[golang]工作池workerpool'
date: 2016-03-15 18:19:32
tags: golang
---

在并发量比较高的场景中, 服务程序既要快速处理请求, 也要限制并行处理的gorontine量, 否则会造成系统资源的浪费或负载过高。

创建一个goroutine工作池, 是一个比较合理的解决方案。

### 示例1
<!--more-->
```golang
package main

import (
	"fmt"
	"log"
	"time"
)

const (
	// PoolSize defines the pool size
	PoolSize = 10
)

// Jobs jobs channel
var Jobs chan string = make(chan string, 30)
var Result chan string = make(chan string, 30)

// Working do working
func Working(workID int) {
	log.Printf("Initiate worker %v...", workID)
	for j := range Jobs {
		log.Printf("work id: %v, receive msg: %v", workID, j)
		Result <- fmt.Sprintf("%v is done...", j)
		time.Sleep(time.Second * 1)
	}
}

// InitWorkerPool initialize worker pool
func InitWorkerPool() {
	for j := 0; j < PoolSize; j++ {
		go Working(j)
	}
}

func main() {
	InitWorkerPool()
	// to generate jobs
	time.Sleep(time.Second * 2)
	for i := 0; i < 20; i++ {
		job := fmt.Sprintf("JobID %v", i)
		Jobs <- job
	}
	close(Jobs)

	for i := 0; i < 20; i++ {
		log.Printf("working result: %v", <-Result)
	}
}
```
运行结果:
```bash
2016/04/12 14:18:58 Initiate worker 0...
2016/04/12 14:18:58 Initiate worker 1...
2016/04/12 14:18:58 Initiate worker 2...
2016/04/12 14:18:58 Initiate worker 3...
2016/04/12 14:18:58 Initiate worker 4...
2016/04/12 14:18:58 Initiate worker 5...
2016/04/12 14:18:58 Initiate worker 6...
2016/04/12 14:18:58 Initiate worker 7...
2016/04/12 14:18:58 Initiate worker 8...
2016/04/12 14:18:58 Initiate worker 9...
2016/04/12 14:19:00 work id: 9, receive msg: JobID 0
2016/04/12 14:19:00 work id: 0, receive msg: JobID 1
2016/04/12 14:19:00 work id: 1, receive msg: JobID 2
2016/04/12 14:19:00 work id: 2, receive msg: JobID 3
2016/04/12 14:19:00 work id: 3, receive msg: JobID 4
2016/04/12 14:19:00 work id: 4, receive msg: JobID 5
2016/04/12 14:19:00 work id: 5, receive msg: JobID 6
2016/04/12 14:19:00 work id: 6, receive msg: JobID 7
2016/04/12 14:19:00 work id: 7, receive msg: JobID 8
2016/04/12 14:19:00 work id: 8, receive msg: JobID 9
2016/04/12 14:19:00 working result: JobID 0 is done...
2016/04/12 14:19:00 working result: JobID 1 is done...
2016/04/12 14:19:00 working result: JobID 2 is done...
2016/04/12 14:19:00 working result: JobID 3 is done...
2016/04/12 14:19:00 working result: JobID 4 is done...
2016/04/12 14:19:00 working result: JobID 5 is done...
2016/04/12 14:19:00 working result: JobID 6 is done...
2016/04/12 14:19:00 working result: JobID 7 is done...
2016/04/12 14:19:00 working result: JobID 8 is done...
2016/04/12 14:19:00 working result: JobID 9 is done...
2016/04/12 14:19:01 work id: 8, receive msg: JobID 10
2016/04/12 14:19:01 work id: 9, receive msg: JobID 11
2016/04/12 14:19:01 work id: 0, receive msg: JobID 12
2016/04/12 14:19:01 work id: 1, receive msg: JobID 13
2016/04/12 14:19:01 work id: 2, receive msg: JobID 14
2016/04/12 14:19:01 work id: 3, receive msg: JobID 15
2016/04/12 14:19:01 work id: 4, receive msg: JobID 16
2016/04/12 14:19:01 work id: 5, receive msg: JobID 17
2016/04/12 14:19:01 work id: 6, receive msg: JobID 18
2016/04/12 14:19:01 work id: 7, receive msg: JobID 19
2016/04/12 14:19:01 working result: JobID 10 is done...
2016/04/12 14:19:01 working result: JobID 11 is done...
2016/04/12 14:19:01 working result: JobID 12 is done...
2016/04/12 14:19:01 working result: JobID 13 is done...
2016/04/12 14:19:01 working result: JobID 14 is done...
2016/04/12 14:19:01 working result: JobID 15 is done...
2016/04/12 14:19:01 working result: JobID 16 is done...
2016/04/12 14:19:01 working result: JobID 17 is done...
2016/04/12 14:19:01 working result: JobID 18 is done...
2016/04/12 14:19:01 working result: JobID 19 is done...
```
由上可以见, 程序中同时最多有10个工作任务在运行。

### 示例2, 更规范的用法
我们引入同步锁, 并把工作池定义成一个结构体WorkPool， 包含poolSize和tasks成员, 并实现WorkPool.Run方法:
```golang
package main

import (
	"fmt"
	"log"
	"sync"
	"time"
)

// WorkPool worker pool definition
type WorkPool struct {
	poolSize int
	tasks    chan string
}

// Run initialize worker pool, ready to run tasks
func (wp *WorkPool) Run() {
	var wg sync.WaitGroup
	for i := 0; i < wp.poolSize; i++ {
		wg.Add(1)
		taskID := i
		go func() {
			log.Printf("Initiate worker %v...", taskID)
			for j := range wp.tasks {
				log.Printf("work id: %v, receive msg: %v", taskID, j)
				time.Sleep(time.Second * 1)
			}
			wg.Done()
		}()
	}
	wg.Wait()
	log.Println("WorkPool is destroyed.")
}

func main() {
	tasks := make(chan string, 20)

	// to generate jobs
	go func() {
		time.Sleep(time.Second * 2)
		for i := 0; i < 30; i++ {
			job := fmt.Sprintf("jobID %v", i)
			tasks <- job
		}
		// if not close tasks, will cause deadlock
		close(tasks)
	}()

	wp := WorkPool{
		poolSize: 10,
		tasks:    tasks,
	}
	wp.Run()
}
```
运行结果:
```bash
2016/04/12 15:01:03 Initiate worker 9...
2016/04/12 15:01:03 Initiate worker 0...
2016/04/12 15:01:03 Initiate worker 1...
2016/04/12 15:01:03 Initiate worker 2...
2016/04/12 15:01:03 Initiate worker 3...
2016/04/12 15:01:03 Initiate worker 4...
2016/04/12 15:01:03 Initiate worker 5...
2016/04/12 15:01:03 Initiate worker 6...
2016/04/12 15:01:03 Initiate worker 7...
2016/04/12 15:01:03 Initiate worker 8...
2016/04/12 15:01:05 work id: 8, receive msg: jobID 0
2016/04/12 15:01:05 work id: 9, receive msg: jobID 1
2016/04/12 15:01:05 work id: 0, receive msg: jobID 2
2016/04/12 15:01:05 work id: 1, receive msg: jobID 3
2016/04/12 15:01:05 work id: 2, receive msg: jobID 4
2016/04/12 15:01:05 work id: 3, receive msg: jobID 5
2016/04/12 15:01:05 work id: 4, receive msg: jobID 6
2016/04/12 15:01:05 work id: 5, receive msg: jobID 7
2016/04/12 15:01:05 work id: 6, receive msg: jobID 8
2016/04/12 15:01:05 work id: 7, receive msg: jobID 9
2016/04/12 15:01:06 work id: 7, receive msg: jobID 10
2016/04/12 15:01:06 work id: 8, receive msg: jobID 11
2016/04/12 15:01:06 work id: 9, receive msg: jobID 12
2016/04/12 15:01:06 work id: 0, receive msg: jobID 13
2016/04/12 15:01:06 work id: 1, receive msg: jobID 14
2016/04/12 15:01:06 work id: 2, receive msg: jobID 15
2016/04/12 15:01:06 work id: 3, receive msg: jobID 16
2016/04/12 15:01:06 work id: 4, receive msg: jobID 17
2016/04/12 15:01:06 work id: 5, receive msg: jobID 18
2016/04/12 15:01:06 work id: 6, receive msg: jobID 19
2016/04/12 15:01:07 work id: 6, receive msg: jobID 20
2016/04/12 15:01:07 work id: 7, receive msg: jobID 21
2016/04/12 15:01:07 work id: 8, receive msg: jobID 22
2016/04/12 15:01:07 work id: 9, receive msg: jobID 23
2016/04/12 15:01:07 work id: 0, receive msg: jobID 24
2016/04/12 15:01:07 work id: 1, receive msg: jobID 25
2016/04/12 15:01:07 work id: 2, receive msg: jobID 26
2016/04/12 15:01:07 work id: 3, receive msg: jobID 27
2016/04/12 15:01:07 work id: 4, receive msg: jobID 28
2016/04/12 15:01:07 work id: 5, receive msg: jobID 29
2016/04/12 15:01:08 WorkPool is destroyed.
```
由上可见, 同时最多只有10个任务在执行；调用close(tasks)后, 所有工作任务完成, 工作池销毁, 主进程退出。

