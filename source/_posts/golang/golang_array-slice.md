---
layout: '[golang]array-slice'
title: golang数组与切片
date: 2017-01-30 09:00:18
tags: golang
---

### array类型
#### 1)基础特性
- array为固定长度的数组, 其内存分配为连续的, 使用前需确定长度;
- 数组为值类型, 赋值操作的新变量是原数组的一份完整拷贝;
- 作为函数的传递参数, 实际也是数组的一份拷贝, 效率也就比传递指针低;
- 数组长度也是Type一部分, 如[4]int和[2]int类型不一样.

<!--more-->

#### 2)声明与初始化
```golang
package main
import "fmt"

// 只声明, 不作初始化
var a1 [3]int         //一维, [0 0 0]
var a2 [2][2]int      //二维, [[0,0],[0,0]]

// 声明并初始化
var b1 [5]int = [5]int{1,2,3,4,5}
var b2 [4]string = [4]string{"Red", "Blue", "Green", "Yellow"}
var b3 [2][2]int = [2][2]int{[2]int{5,6}, [2]int{7,8}}
// 不能在顶层进行快速声明和初始化, 需用: var b4 [3]string = ...
//b4 := [3]string{"a", "b", "c"}

// 几种常用初始化方法(在函数内使用)
func arrInFunc(){
    a := [3]int{1,2,3}    // 所有元素赋值
    b := [5]int{1,2,3}    // 前三个元素赋值，其他默认0
    c := [15]int{10:3}    // 指定第11个元素初始化为3，其他默认0
    d := [...]int{4,5,6}  // 编译器自动推断长度
    e := [...]int{0:1, 1:2, 5:7} //自动推断长度
    fmt.Printf("%v\n", a)
    fmt.Printf("%v\n", b)
    fmt.Printf("%v\n", c)
    fmt.Printf("%v\n", d)
    fmt.Printf("%v\n", e)
}

func main(){
    arrInFunc()
}
```
main.go演示效果如下:
```bash
root@XIAOMO:~/gopro# ./main
[1 2 3]
[1 2 3 0 0]
[0 0 0 0 0 0 0 0 0 0 3 0 0 0 0]
[4 5 6]
[1 2 0 0 0 7]
```

#### 3)元素访问
main.go示例
```golang
package main

func accessElem() {
    array := [3]int{1, 3, 9}

    // 通过下标访问
    for i:=0; i < len(array); i++ {
        fmt.Println(i, array[i])
    }

    // 迭代方式访问
    for i, v := range array {
        fmt.Println(i, v)
    }

    // 使用new创建数组,零值填充, 返回数组指针
    p := new([5]int)
    fmt.Println(*p)
}

func main(){
    accessElem()
}
```

示例演示效果:
```bash
root@XIAOMO:~/gopro# ./main
0 1
1 3
2 9
0 1
1 3
2 9
[0 0 0 0 0]
```

#### 4)在函数中传递数组
和C++类似, 函数参数传递中可以传值或传指针:

```golang
// 传值, 每次调用foo1, 系统将分配16字节内存在栈上
// 函数运行结束时, 会弹栈并释放16字节内存
func foo1(arr [16]int) {
   // ...
}
var a [16]int
foo1(a)

// 传指针, 每次调用foo2, 系统将只分配指针需要的内存空间
func foo2(arr *[16]int){
    // ...
}
var b [16]int
foo2(&b)
```

### slice类型
#### 1)基础特性
+ slice是一种动态数组, 可以认为是指向数组的指针; 但其并不只是指针，本身有其数据结构, 该结构包含三个元素:
    - 指向原生数组的指针(pointer)
    - 数组切片的元素个数(len)
    - 数组切片已分配的空间(cap)
+ slice作为一个引用类型, 声明是不需要指定长度;
+ 增长操作通过内建方法append实现, 内部实现自动扩容.

#### 2)创建和初始化
main.go
```golang
package main

func createSlice() {
    // 通过array创建slice, 用法神似python
    arr := [5]int{1,2,3,4,5}
    sli1 := arr[:3]  // 切出前三个元素
    sli2 := arr[:4]  // 切出前四个元素
    sli1[1] = 8
    sli2[1] = 9
    // 将打印 9 9 9, 可见修改的是同一个元素
    println(arr[1], sli1[1], sli2[1])

    // 通过make创建slice
    sli3 := make([]int, 3)
    sli4 := make([]int, 4, 8) //初始4个元素，预留8个元素的空间
    sli5 := []int{1,2,3,4,5}  //初始化赋值
    fmt.Printf("%v\n", sli3)
    fmt.Printf("%v\n", sli4)
    fmt.Printf("%v\n", sli5)
}

func main(){
    createSlice()
}
```
#### 3)元素访问
```golang
package main

func useSlice() {
    slice := []int{10, 20, 30, 40, 50}
    for index, value := range slice {
        fmt.Printf("Index: %d  Value: %d\n", index, value)
    }

    for i := 0; i < len(slice); i++ {
        fmt.Printf("Index: %d  Value: %d\n", i, slice[i])
    }
}

func main(){
    useSlice()
}
```
#### 4)在函数间传递slice
由于slice是指向底层数组的指针, 在函数间传递slice是开销很小的。
在64位机器中, slice对象占24个字节, 三个元素分别占8个字节。
作为参数在函数中传递的方式和数组类似。

### 关于new与make的区别探讨
主要区别:

- new可以用来创建各种类型对象, 也即是各类型的空间分配;
- make用来处理内建类型(slice, channel, map等)的内存分配.
