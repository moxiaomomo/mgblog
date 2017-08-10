---
title: '[golang]json-rpc'
date: 2016-03-12 18:09:09
tags: golang
---

#### 关于JSON-RPC
JSON-RPC是一个远程调用协议, 还是轻量级的, 简单易用。
在请求远程调用的时候, 我们可以这样定义请求体:
```json
{
    "jsonrpc": "2.0",
    "method": "login",
    "params": {"name":"momo", "passwd":"xxxxxx"},
    "id": 0
}
```
<!--more-->
关于其中几个参数, 有如下说明:
```
jsonrpc: 协议的版本号(1.0版本是不带该参数, 2.0版带该参数并且值为"2.0")
method:  所要调用的方法名
params:  所调用方法接收的参数(/列表)
id:      当次请求的标识码(服务端响应体内应包含相同的id)
```
更详细的说明, 可以参考这里: http://wiki.geekdream.com/Specification/json-rpc_2.0.html

#### Golang JSON-RPC

golang中有rpc包:<br>
***net/rpc*** <br>实现了最基本的rpc调用，默认通过HTTP协议传输gob数据来实现远程调用。<br>
***net/rpc/jsonrpc*** <br>实现了JSON-RPC协议， 也就实现了对json数据的序列化和反序列化。

- 服务端示例

```golang
package main

import (
	"errors"
	"log"
	"net"
	"net/rpc"
	"net/rpc/jsonrpc"
)

// RpcObj和ReplyObj为服务端/客户端之间传输的数据结构体
// C/S双方都能处理这两个数据类型
type RpcObj struct {
	A, B int
}

type ReplyObj struct {
	C int
}

type Arith int

// 服务端的响应结构体
type ArithAddResp struct {
	Id     interface{} `json:"id"`
	Result ReplyObj    `json:"result"`
	Error  interface{} `json:"error"`
}

// 符合 func (t T) funname(t1 *T1, t2 *T2) error 类型的方法都可以注册到rpc中
func (t *Arith) Add(args *RpcObj, reply *ReplyObj) error {
	reply.C = args.A + args.B
	return nil
}

func (t *Arith) Mul(args *RpcObj, reply *ReplyObj) error {
	reply.C = args.A * args.B
	return nil
}

func (t *Arith) Div(args *RpcObj, reply *ReplyObj) error {
	if args.B == 0 {
		return errors.New("divide by zero")
	}
	reply.C = args.A / args.B
	return nil
}

func (t *Arith) Error(args *RpcObj, reply *ReplyObj) error {
	panic("ERROR")
}

func main() {
	arith := new(Arith)
	// 新建rpc server
	server := rpc.NewServer()
	// 注册handler处理器
	server.Register(arith)
	// 监听8080端口
	l, e := net.Listen("tcp", ":8080")
	if e != nil {
		log.Fatal("listen error:", e)
	}

	log.Println("RPC server started...")
	for {
		// 接收客户端请求
		conn, err := l.Accept()
		if err != nil {
			log.Fatal(err)
		}

		// 在goroutine中处理请求
		// 通过http.Conn创建一个jsonrpc编码器, 并将其传递到rpc编码器
		go server.ServeCodec(jsonrpc.NewServerCodec(conn))
	}
	log.Println("RPC server is shutdown...")
}

```

- 客户端示例

```golang
package main

import (
	"log"
	"net"
	"net/rpc/jsonrpc"
)

type RpcObj struct {
	A, B int
}

type ReplyObj struct {
	C int
}

type Arith int

type ArithAddResp struct {
	Id     interface{} `json:"id"`
	Result ReplyObj    `json:"result"`
	Error  interface{} `json:"error"`
}

func main() {
	conn, err := net.Dial("tcp", "localhost:8080")

	if err != nil {
		panic(err)
	}
	defer conn.Close()

	c := jsonrpc.NewClient(conn)

	var reply ReplyObj
	var args *RpcObj
	for i := 5; i >= 0; i-- {
		// 往RPC调用传递参数
		args = &RpcObj{5, i}

		// 远程调用Arith.Mul方法
		err = c.Call("Arith.Mul", args, &reply)
		if err != nil {
			log.Fatal("Exited as arith error:", err)
		}
		log.Printf("Arith: %d * %d = %v\n", args.A, args.B, reply.C)

		// 远程调用Arith.Div方法
		err = c.Call("Arith.Div", args, &reply)
		if err != nil {
			log.Fatal("Exited as arith error:", err)
		}
		log.Printf("Arith: %d / %d = %v\n", args.A, args.B, reply.C)

		// 华丽的分割线
		log.Printf("\033[33m%s\033[m\n", "---------------")
	}
}
```

- 示例效果

```bash
root@XIAOMO:/data/apps/demo# go run rpccli.go
2016/03/28 16:17:41 Arith: 5 * 5 = 25
2016/03/28 16:17:41 Arith: 5 / 5 = 1
2016/03/28 16:17:41 ---------------
2016/03/28 16:17:41 Arith: 5 * 4 = 20
2016/03/28 16:17:41 Arith: 5 / 4 = 1
2016/03/28 16:17:41 ---------------
2016/03/28 16:17:41 Arith: 5 * 3 = 15
2016/03/28 16:17:41 Arith: 5 / 3 = 1
2016/03/28 16:17:41 ---------------
2016/03/28 16:17:41 Arith: 5 * 2 = 10
2016/03/28 16:17:41 Arith: 5 / 2 = 2
2016/03/28 16:17:41 ---------------
2016/03/28 16:17:41 Arith: 5 * 1 = 5
2016/03/28 16:17:41 Arith: 5 / 1 = 5
2016/03/28 16:17:41 ---------------
2016/03/28 16:17:41 Arith: 5 * 0 = 0
2016/03/28 16:17:41 Exited as arith error:divide by zero
exit status 1
```
