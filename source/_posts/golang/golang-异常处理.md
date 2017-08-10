---
title: '[golang]异常处理'
date: 2016-03-10 13:53:30
tags: golang
---

### 关于异常处理

golang并不支持try-catch-finally机制，而是通过panic和defer函数来处理运行时异常。<br>
一般情况下，golang中使用多值来返回错误；个别情况下才需要用到Exception异常处理: defer, panic, recover。

<!--more-->
示例:
```golang
package main

import (
    "fmt"
)

// 一个函数可以声明多个defer, 按FILO规则执行
func SimpleExceptionSample() {
    defer func() {
        // recover()可以获取panic的参数
        if err := recover(); err != nil {
            fmt.Println("panic_1 msg: ", err)
        }
    }()

    defer func() {
        fmt.Println("panic_2.")
    }()

    defer func() {
        fmt.Println("panic_3.")
    }()

    panic("something wrong.")
}

func main() {
    SimpleExceptionSample()
}
```
运行结果:
```bash
root@XIAOMO:~/gopro# go build excep.go
root@XIAOMO:~/gopro# ./excep
panic_3.
panic_2.
panic_1 msg:  something wrong.
```

### defer的三个特性
defer在函数返回之前执行, 其有三个明显特性:

+ defer表达式中参数变量的值在定义时就已经确定
```golang
func a() {
    i := 1
    defer fmt.Println("i=", i)
    i = i+1
    return
}
```
这样会输出 i=1而非i=2

+ defer表达式的执行顺序遵循FILO(参考第一个示例)
+ defer表达式能修改函数中的命名返回参数值
```golang
func c() (i int) {
    defer func() { i++ }()
    return i
}
```
这样返回值为1而非0(命名返回参数的默认值为0)。

