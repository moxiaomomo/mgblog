---
title: 'golang笔试题整理'
date: 2017-11-14 08:44:28
tags: golang
---

```golang
package main

import "fmt"

func f1() (result int) {
	defer func() {
		result++
	}()
	return 0
}

func f2() (r int) {
	t := 3
	defer func() {
		t = t + 3
	}()
	return t
}

func f3() (r int) {
	defer func(r int) {
		r = r + 2
	}(r)
	return 1
}

func main() {
	fmt.Println(f1())
	fmt.Println(f2())
	fmt.Println(f3())
}

// 考点：defer的一些坑
// return xxx可以改写成一下规则：
// 返回值 = xxx
// 调用defer函数
// 空的return

// 输出：
// 1
// 3
// 1
```
<!--more-->

```golang
package main

import "fmt"

func loop() {
	for i := 0; i < 10; i++ {
		fmt.Printf("%d ", i)
	}
}

func main() {
	go loop()
	loop()
}

// 考点: 线程的执行顺序
// 主线程首先退出，goroutine无法执行

// 输出:
// 0 1 2 3 4 5 6 7 8 9
```

```golang
package main

import "fmt"

var ch1 chan int = make(chan int)
var ch2 chan int = make(chan int)

func worker1(s string) {
	fmt.Println(s)
	ch1 <- <-ch2
}

func worker2(s string) {
	ch1 <- <-ch2
	fmt.Println(s)
}

func main() {
	go worker1("work hard.")
	go worker2("just for fun!")
	<-ch1
}

// 考点：channel的执行顺序
// ch1 <- <-ch2会引起当前goroutine阻塞,
// <-ch1 会引起主线程阻塞,
// 所有goroutine都在阻塞，引起死锁

// 输出:
// work hard.
// fatal error: all goroutines are asleep - deadlock!
```

```golang
package main

type student struct {
	Name string
	Age  int
}

func main() {
	m := make(map[string]*student)
	stus := []student{
		{Name: "zhao", Age: 24},
		{Name: "chen", Age: 23},
		{Name: "wang", Age: 22},
	}
	for _, stu := range stus {
		m[stu.Name] = &stu
	}

	for k, v := range m {
		println(k, "--", v.Name)
	}
}

// 考点：遍历与指针
// m[stu.Name] = &stu： 每次赋值的&stu其实指向同一个地址,
// 最后打印的值为最后一次遍历的student对象的值。

// 输出:
// zhao -- wang
// chen -- wang
// wang -- wang
```

```golang
package main

import "fmt"

type People struct{}
type Teacher struct {
	People
}

func (p *People) Walk() {
	fmt.Println("People Walk")
	p.Work()
}

func (p *People) Work() {
	fmt.Println("People Work")
}

func (t *Teacher) Work() {
	fmt.Println("Teacher Work")
}

func main() {
	t := Teacher{}
	t.Walk()
}

// 考点: golang的组合继承
// 虽然被组合的People类的方法升级成为外部Teacher组合类型的方法，
// 但p.Work()调用时接收者仍然是People, 因此打印"People Work"

// 输出:
// People Walk
// People Work
```

```golang
package main

func test() []func()  {
    var funs []func()
    for i:=0;i<2 ;i++  {
        funs = append(funs, func() {
            println(&i,i)
        })
    }
    return funs
}

func main(){
    funs:=test()
    for _,f:=range funs{
        f()
    }
}

// 考点：闭包延迟求值

// 输出：
// 0xc42000e110 2
// 0xc42000e110 2
```

```golang
package main

import (
    "fmt"
    "reflect"
)

func main1()  {
    defer func() {
       if err:=recover();err!=nil{
           fmt.Println(err)
       }else {
           fmt.Println("fatal")
       }
    }()

    defer func() {
        panic("defer panic")
    }()
    panic("panic")
}

// 考点：panic仅有最后一个被捕获

// 输出：
// defer panic
```

```golang
package main

import (
	"sync"
)

type UserAges struct {
	ages map[string]int
	sync.Mutex
}

func (ua *UserAges) Add(name string, age int) {
	ua.Lock()
	defer ua.Unlock()
	ua.ages[name] = age
}

func (ua *UserAges) Get(name string) int {
	if age, ok := ua.ages[name]; ok {
		return age
	}
	return -1
}

func main() {
	ua := UserAges{}
	ua.Get("yourname")
}

// 考点：map线程安全
// 有可能出现：fatal error: concurrent map read and map write
// 修改， Get方法加锁：
//func (ua *UserAges) Get(name string) int {
//    ua.Lock()
//    defer ua.Unlock()
//    if age, ok := ua.ages[name]; ok {
//        return age
//    }
//    return -1
//}
```

```golang
package main

import (
	"fmt"
)

type People interface {
	Speak(string) string
}

type Stduent struct{}

func (stu *Stduent) Speak(name string) (talk string) {
	if name == "xiaoming" {
		talk = "xiaoming is a good boy."
	} else {
		talk = "who's that guy?"
	}
	return
}

func main() {
	var peo People = Stduent{}
	fmt.Println(peo.Speak("xiaoming"))
}

// 考点: 接口实现与类型检查
// 对于赋值给接口的类型，会有严格的类型检查，不允许做A到A*的转换
// 类型A的方法集（method set）是类型*A的一个子集

// 输出:
// ./p7.go:23: cannot use Stduent literal (type Stduent) as type People in assignment:
//	Stduent does not implement People (Speak method has pointer receiver)

```
