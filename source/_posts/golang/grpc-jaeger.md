---
title: grpc-jaeger调用链跟踪模块
date: 2018-06-17 17:04:05
tags: ["golang","tracing"]
---

# grpc-jaeger说明

具体源码可参考[我的github源码][1]

grpc-jaeger是基于Go的针对gRPC的一种拦截器实现，用于结合jaeger来实现rpc调用链跟踪；可用于集成到微服务的分布式trace功能中。

grpc包中对外暴露了两个接口：grpc.UnaryClientInterceptor及grpc.UnaryServerInterceptor， 只要将这两个函数重写即可以自定义拦截器，从而注入jaeger相关跟踪代码。

<!--more-->

# grpc-jaeger源依赖

```
google.golang.org/grpc
github.com/opentracing/opentracing-go
github.com/uber/jaeger-client-go
```

# grpc-jaeger使用示例

- 在grpc client端，添加grpc-jaeger

```golang
import (
	"fmt"
	"os"
	
	gtrace "github.com/moxiaomomo/grpc-jaeger"
	"google.golang.org/grpc"
)

func rpcCli(dialOpts []grpc.DialOption) {
		conn, err := grpc.Dial("127.0.0.1:8001", dialOpts...)
		if err != nil {
			fmt.Printf("grpc connect failed, err:%+v\n", err)
			return
		}
		defer conn.Close()
		
		// TODO: do some rpc-call
		// ...
}

func main() {
		dialOpts := []grpc.DialOption{grpc.WithInsecure()}
		tracer, _, err := gtrace.NewJaegerTracer("testCli", "127.0.0.1:6831")
		if err != nil {
			fmt.Printf("new tracer err: %+v\n", err)
			os.Exit(-1)
		}

		if tracer != nil {
			dialOpts = append(dialOpts, gtrace.DialOption(tracer))
		}
		// do rpc-call with dialOpts
		rpcCli(dialOpts)
}
```

- 在grpc server端，添加grpc-jaeger

```golang
import (
	"fmt"
	"os"
	
	gtrace "github.com/moxiaomomo/grpc-jaeger"
	"google.golang.org/grpc"
)

func main() {
	var servOpts []grpc.ServerOption
	tracer, _, err := gtrace.NewJaegerTracer("testSrv", "127.0.0.1:6831")
	if err != nil {
		fmt.Printf("new tracer err: %+v\n", err)
		os.Exit(-1)
	}
	if tracer != nil {
		servOpts = append(servOpts, gtrace.ServerOption(tracer))
	}
	svr := grpc.NewServer(servOpts...)
	// TODO: register some rpc-service to grpc server
	
	ln, err := net.Listen("tcp", "127.0.0.1:8001")
	if err != nil {
		os.Exit(-1)
	}
	svr.Serve(ln)
}
```

# 结合jaeger服务进行测试

## 1）部署jaeger

假设在某个节点已部署好jaeger测试服务，<br>
jaeger-agent地址为`192.168.1.100:6831`，<br>
jaeger-ui地址为`http://192.168.1.100:16686/`。

## 2）运行测试

具体测试用例在wrapper_test.go中，可直接在该文件当前目录中运行`go test`

```bash
test@MG:/yourpath/grpc-jaeger$ go test
2018/06/17 15:06:05 Initializing logging reporter
2018/06/17 15:06:06 Initializing logging reporter
SayHello Called.
2018/06/17 15:06:06 Reporting span 107bf923fcfc238e:1f607766f1329efd:107bf923fcfc238e:1
2018/06/17 15:06:06 Reporting span 107bf923fcfc238e:107bf923fcfc238e:0:1
call sayhello suc, res:message:"Hi im tester\n"
PASS
ok  	github.com/moxiaomomo/grpc-jaeger	3.004s
```

## 3）查看jaegerUI
打开`http://192.168.1.100:16686/`后查询对应Service， 可看到以下跟踪结果：![jaegerUI][2]

  [1]: http://github.com/moxiaomomo/grpc-jaeger
  [2]: https://github.com/moxiaomomo/grpc-jaeger/blob/master/jaegerui.png?raw=true
