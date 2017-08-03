---
title: golang之map/set类型
date: 2017-02-01 19:13:13
tags:
---

### map类型
#### 1. 基础特性
- map是一种无序的键值对的集合; 所以可以类似数组/slice一样进行迭代
- map的值可以使内建类型, 也可以是struct类型
- 内部使用hash表实现, map的hash表包含了一个collection of buckets（桶集合）

<!--more-->

#### 2. 声明与初始化
```golang
package main

import (
    "fmt"
)

// map[keyType]valueType
func initMap() {
    // 通过make方法创建
    dict := make(map[string]int)
    dict["age"] = 18

    // 直接创建
    dict2 := map[string]string{"name":"xiaoming", "phone":"135xxx"}
    dict2["addr"] = "Guangzhou"

    fmt.Printf("%v\n", dict2)
}


func main() {
    initMap()
}
```

#### 3. 元素访问
```golang
package main

import (
    "fmt"
)

type Student struct {
    name string
    grade int
}

func useMap() {
    //使用前应该先初始化, 否则panic报错
    // var map1 map[string]string
    // map1["a"] = "b" // will panic

    map1 := make(map[string]Student)
    map1["s1"] = Student{name:"xiaomo", grade:1}
    fmt.Printf("%v\n", map1)
}

func main() {
    useMap()
}

```

#### 4. 在函数中传递map

在函数间传递map对象, 是传递引用而不是拷贝; 因此在函数中对map进行了修改, 引用到它的地方也会相应修改
```golang
package main

import (
    "fmt"
)

type Student struct {
    name string
    grade int
}

func useMap() {
    map1 := make(map[string]Student)
    map1["s1"] = Student{name:"xiaomo", grade:1}
    // 作为函数参数传递
    printMap(map1)
}

func printMap(m map[string]Student) {
    fmt.Printf("currentMap: %v\n", m)
}

func main() {
    useMap()
}

```

### Set类型

golang没有内置Set类型, 可以自定义实现。

