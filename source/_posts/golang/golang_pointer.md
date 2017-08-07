---
title: '[golang]指针类型'
date: 2016-04-05 19:35:16
tags: golang
---

### 关于golang指针类型
Golang保留了指针类型,支持指针操作。<br>
可以使用操作符&取变量地址，使用操作符*通过指针变量间接访问目标对象。<br>
unsafe.Pointer可以与任意指针类型转换, 可以将unsafe.Pointer转换为uintptr,然后做指针运算。<br>
uintptr类型可以转换为整数。
<!--more-->
### 指针使用示例

```golang
package main

import (
    "fmt"
    "unsafe"
)

type User struct {
    Id int
    Name string
}

func main(){
    i := 10
    var p *int = &i  //取地址
    fmt.Println(*p) // 取值

    user := &User{1, "xiaomo"}
    user.Id = 100  //直接对指针对象的成员赋值
    fmt.Println(user)

    user2 := *user  //拷贝对象
    user2.Id = 200
    user2.Name = "xiaomomo"
    fmt.Println(user, user2)

    up := unsafe.Pointer(p)  // 转为通用指针类型
    uptr := uintptr(up)      // 转为uintptr指针类型
    fmt.Println(up, uptr)
}
```

###运行效果如下:
```bash
root@XIAOMO:~/gopro# go build main.go
root@XIAOMO:~/gopro# ./main
10
&{100 xiaomo}
&{100 xiaomo} {200 xiaomomo}
0xc210000070 833492090992
```
