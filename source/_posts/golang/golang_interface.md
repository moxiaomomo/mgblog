---
title: '[golang]interface'
date: 2016-01-02 15:22:43
tags: golang
---


第一次接触golang, 应该是2013那年在某沙龙听许大神在介绍golang特性。<br>
那时候据说interface可以说是golang中最具特色、设计精妙的特性之一。<br>
现在计划写个demo， 重新学习一下interface的设计模式。

### 什么是interface
- 在golang中, interface简单可认为是一组函数的组合, 通过interface可以定义对象的一组行为。
- interface定义了一组方法, 如果某个对象实现了某个interface的所有方法， 则我们可以认为这个对象实现了这个接口。
- 由此可见, 不像C++、java， golang实现接口并不需要显式的implements。。。

<!--more-->

### interface的一些基础特性
- interface类型不包含任何的成员变量, 只有方法;
- interface类型进行类型转换时, 默认返回的是对象拷贝; 如果需要修改源对象, 需使用指针;
- interface类型可以嵌套, 但不支持两个对象互相嵌套.


### 接口的定义与实现
一个例子
```golang
package main

import (
    "fmt"
)

type Human interface {
    GetName() string
    GetAge() int
    GetGender() string
}

// 定义struct Employee
type Employee struct {
    name   string
    age    int
    salary int
    gender string
}

// 定义struct Employee 的方法
func (self *Employee) GetName() string {
    return self.name
}

func (self *Employee) GetAge() int {
    return self.age
}

func (self *Employee) GetGender() string{
    return self.gender
}

func (self *Employee) GetSalary() int {
    return self.salary
}

// 参数为Human类型
func printName(p Human) {
    name := p.GetName()
    fmt.Printf("name:%v\n", name)
}

func main() {
    varEmployee := Employee{
        name:   "xiaomo",
        age:    30,
        salary: 10000000,
        gender: "Male",
    }
    // 访问GetSalary方法
    s := varEmployee.GetSalary()
    fmt.Printf("salary:%v\n", s)

    // 子集转为超集类型Human, 作为参数传入函数中
    var p Human = &varEmployee
    printName(p)
}
```
演示效果:
```bash
root@XIAOMO:~/gopro# go build interface.go
root@XIAOMO:~/gopro# ./interface
salary:10000000
name:xiaomo
```
上例中, 定义了接口Human, 内部有三个方法: GetName(), GetAge(), GetGender(); <br>
结构Employee除了实现这三个方法外, 还实现了GetSalary()方法。 因此, 对象Employee实现Human接口。

Employee可以转换为Human类型, 作为形参传入printName()函数; 实现了多态调用。

### 接口的嵌套
一个接口可以嵌套另一个接口，根据上面的代码再修改一下， 修改部分如下:
```golang
// 嵌套接口Human, 那么User除了有PrintInfo()外， 也拥有了Human的所有方法
type User interface {
    Human
    PrintInfo()
}

// 实现PrintInfo()方法
func (self * Employee) PrintInfo() {
    fmt.Printf("name:%v age:%v salary:%v gender:%v\n",
        self.name, self.age, self.salary, self.gender)
}

func main() {
    varEmployee := Employee{
        name:   "xiaomo",
        age:    50,
        salary: 100000000,
        gender: "Male",
    }
    // 访问GetSalary方法
    s := varEmployee.GetSalary()
    fmt.Printf("salary:%v\n", s)

    // 子集转为超集类型Human, 作为参数传入函数中
    var p Human = &varEmployee
    printName(p)

    // Employee转换为User类型
    var u User = &varEmployee
    // 这时可以调用User的GetName()方法
    fmt.Printf("user_name:%v\n", u.GetName())
    // 调用User的PrintInfo()方法
    u.PrintInfo()
}
```
### 空接口(Any类型)
golang中所有对象都满足空接口interface{}, 因此interface{}可以指向任何类型的对象。

```golang
var i1 interface{} = 100      // 将int类型赋值给interface{}
var i2 interface{} = "ok"     // 将string类型赋值给interface{}
var i3 interface{} = &i2      // 将*interface{}类型赋值给interface{}
var i4 interface{} = struct{ X int }{1}   // 将struct类型赋值给interface{}
var i5 interface{} = &struct{ X int }{1}  // 将*struct类型赋值给interfae{}
```
当一个方法需要接受任意类型的对象时, 我们将其参数声明为interface{}空类型。如fmt库中的各种print方法就使用了这种方式:
```golang
func Printf(fmt string, args ...interface{})
func Println(args ...interface{})
```

### 总结
golang的接口和其他语言的接口区别还是比较大, 显得别具一格。
其他大部分语言的接口是侵入式的， 需要显式的implement；而golang是非侵入式的, 实现某个接口并不需要从该接口继承，而只需要实现该接口的所有方法。
