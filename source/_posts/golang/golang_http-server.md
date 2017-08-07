---
title: '[golang]HTTP webserver'
date: 2016-12-03 14:11:45
tags: golang
---

### 简单示例
golang对http网络模块有比较完备的标准库支持, 如下示例:

<!--more-->

```golang
package main

import (
    "io"
    "net/http"
    "log"
)


func LogicServer(w http.ResponseWriter, req *http.Request) {
    io.WriteString(w, "I'm logic server.")
}


func main() {
    http.HandleFunc("/test", LogicServer)
    err := http.ListenAndServe(":8888", nil)
    if err != nil {
        log.Fatal("ListenAndServe Err:", err)
    }
}
```
运行效果:
```bash
xiaomo@XIAOMO:~$ curl "http://127.0.0.1:8888/test"
I'm logic server.xiaomo
```

### golang http包的执行流程
参考:[Go如何使得Web工作](https://astaxie.gitbooks.io/build-web-application-with-golang/content/zh/03.3.html "http")
![http包运行机制](/img/golang_http.png)

### 结合goroutine处理并发
在golang中, 为了实现高并发与高性能, 处理连接的读写事件使用了goroutine机制。因此, 对于每个请求都能保持独立性, 相互没有阻塞, 能高效响应网络请求事件。

