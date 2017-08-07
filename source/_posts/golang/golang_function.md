---
title: [golang]function
date: 2016-02-07 10:07:46
tags: golang
---

### 基本用法
golang函数是不支持嵌套(但是可以使用匿名函数实现嵌套)、默认参数、重载, 但支持:
- 不定长度的变参
- 多返回值(类似python的返回元组?)
- 显式命名返回值参数
- 匿名函数
- 函数闭包

<!--more-->

以下是一些基本用法示例:
```golang
// 一般用法, 无返回值
func funcName1(input1 type1){
    //do something
}

// 多参数, 一个返回值
func funcName2(input1 type1, input2 type2) (output1 type1) {
    //do something
    return value1
}

// 多参数, 多返回值
func funcName3(input1 type1, input2 type2) (output1 type1, output2 type2) {
    //do something
    return value1, value2
}

// 传入参数同类型
func funcName4(i, j int) int {
    return i*j
}

// 命名返回参数
func funcName5(a, b int) (x, y int) {
    x = a+b
    y = a*b
    return          // 返回 x, y
    // return x, y  // 返回的结果和函数头声明一样: x,y,  可以不写
    // return y, x  // 返回的结果就是y,x 而不是x, y
}

```
### 参数传递
#### 1.传递可变参数
- 可变参数实际为一个slice, 必须作为一个形参放在最后的位置.
- python中变参可用**dict方式传递; golang中需要用 ... 来展开, 否则会被当做一个参数传递

```golang
package main

import (
    "fmt"
)

func testListArg(x int, args ...string) {
    str := ""
    for _, s := range args {
        str += s
    }
    fmt.Printf("%v %v\n", x, str)
}

func main() {
    a := []string{"a", "d", "v", "m"}
    testListArg(5, a...)
    testListArg(6, a[:3]...)
}

```
演示效果:
```bash
root@XIAOMO:~/gopro# go build main.go
root@XIAOMO:~/gopro# ./main
5 advm
6 adv
```
#### 2.传递指针类型
- string, slice, map的传递实际为指针方式传递, 无需取值后显式传递指针
- C/C++指针通过 -> 来获取指针对象成员, golang则通过 . 来实现操作。

```golang
package main

import (
    "fmt"
)

func testPointer(user *User) {
    fmt.Printf("name: %v, id: %v\n", (*user).name, (*user).id)
}

func main() {
    user := User{name:"xiaomo",id:1}
    testPointer(&user)
}
```
#### 3.匿名函数与闭包
匿名方法经常和闭包配套使用
```golang
package main

import (
    "fmt"
)

// 匿名方法
var f1 = func(a,b int) int {
    return (a+b)*2
}

// 匿名方法与闭包
func f2(name string) {
    f_inner := func (x,y int) int {
        return x+y
    }
    sum := f_inner(7, 8)
    fmt.Printf("name:%v sum:%v\n", name, sum)

    // 匿名函数访问外部方法局部变量
    extra := 15
    f_inner2 := func(x,y int) int {
        return x+y+extra
    }
    sum2 := f_inner2(9, 10)
    fmt.Printf("name:%v sum2:%v\n", name, sum2)
}

func main() {
    res := f1(3,4)
    fmt.Printf("%v\n", res)

    f2("xiaomo")
}
```
演示效果:
```golang
root@XIAOMO:~/gopro# go build main.go
root@XIAOMO:~/gopro# ./main
14
name:xiaomo sum:15
name:xiaomo sum2:34

```