---
title: '[golang]channel管道类型'
date: 2016-03-04 09:00:11
tags: golang
---

在unix系统中, 管道是进程间通信的一种方式。那golang的channel和系统的管道有什么区别？

### 什么是channel
官方文档有个channel的定义:


>    A channel provides a mechanism for concurrently executing functions to communicate
>    by sending and receiving values of a specified element type.
>    The value of an uninitialized channel is nil.

channel应该可以理解为: 一种为并行执行而提供函数间通信的机制, 通过传递send和recieve特定类型的值来实现。没有初始化值的channel对象, 默认为nil。
<!--more-->

也就是: 在管道的两端, 一个对象可以在一端进行写/读/读写的操作, 另一个对象可以在另一端进行读/写/读写的操作；能否读写, 取决于管道是否是双向的。默认情况下, 管道是双向的。

### channel的基础特性
- channel是一个作为队列queue的值。 当queue满时, 写操作是阻塞的; 当queue为空时, 读操作是阻塞的
- 向一个nil channel读数据或写数据, 会造成一直阻塞
- 向一个close了的channel写数据, 会引起panic
- 向一个close了的channel读数据, 会立刻获取到一个零值

### 一些用法示例
#### 1.没有缓冲的channel
当我们没有为channel指定容量时, 我们称这个channel是没有缓冲的。
```golang
func test_deadlock() {
     c := make(chan string)
     c <- "test"    // 写入channel
     s := <-c       // 读取channel
     println(s)
}
```
运行效果:
```bash
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.test_deadlock
	/root/gopro/channel.go:10
main.main
	/root/gopro/channel.go:53
```
提示出现了死锁。这是因为处在同一个单线程环境中, 两个goroutine互相等待对方释放资源, 造成双方都无法继续运行下去。Go运行时检测到了死锁并给出了错误报告。指定容量后可解决这个问题:
```golang
c := make(chan string, 3)  // 长度为3
```

#### 2. 使用range来进行读取操作
- range会在close channel后读取完所有数据并自动结束循环
- 有/无缓冲的channel在使用range时效果是一样的
- range方法, 不需要知道channel中消息的准确数目, 使用人性化

使用range示例:
```golang
func main() {
     message := make(chan int)
     count := 3

     go func() {
          for i := 1; i <= count; i++ {
               fmt.Printf("send %v\n", i)
               message <- i
          }
          //完成写入后, 需要关闭channel, 否则主goroutine会因一直等待而死锁
          close(message)
     }()
     time.Sleep(time.Second * 3)

     for msg := range message {
          fmt.Printf("recieve: %v\n", msg)
     }
}
```
运行效果:
```golang
root@XIAOMO:~/gopro# go build channel.go
root@XIAOMO:~/gopro# ./channel
send 1
// 此处等待3秒
recieve: 1
send 2
send 3
recieve: 2
recieve: 3
```

#### 3. 多channel模式
在程序中需要用到多个channel时, 我们可以使用select(类似于switch)来实现对channel的支持。

以下示例中方法CreateMsgChannel()创建了一个channel和goroutine, 并向该channel中分三次写入数据; 在main()方法中通过select来读取三个channel中的数据:
```golang
func CreateMsgChannel(msg string, delay time.Duration) <-chan string {
     c := make(chan string)
     go func() {
          for i := 1; i <= 3; i++ {
               c <- fmt.Sprintf("%s\tcur_loop:%d", msg, i)
               // 模拟处理时间
               time.Sleep(time.Millisecond * delay)
          }
     }()
     return c
}

func main() {
    c1 := CreateMsgChannel("msg in channel 1", 500)
    c2 := CreateMsgChannel("msg in channel 2", 260)
    c3 := CreateMsgChannel("msg in channel 3", 50)

    for i := 1; i <= 9; i++ {
        select {
            case msg := <-c1:
                println(msg)
            case msg := <-c2:
                println(msg)
            case msg := <-c3:
                println(msg)
        }
    }
}
```
运行效果:
```golang
root@XIAOMO:~/gopro# go build channel.go
root@XIAOMO:~/gopro# ./channel
msg in channel 1	cur_loop:1
msg in channel 2	cur_loop:1
msg in channel 3	cur_loop:1
msg in channel 3	cur_loop:2
msg in channel 3	cur_loop:3
msg in channel 2	cur_loop:2
msg in channel 1	cur_loop:2
msg in channel 2	cur_loop:3
msg in channel 1	cur_loop:3
```
由结果可见, 每个channel中的消息都能尽快的被读取, 而不会因为一个channel的阻塞而导致了其他channel的消息无法被读取。