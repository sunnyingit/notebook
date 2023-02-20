Context使用指南


## 概述

Context用于goroutine之间的通信。

Context可以在Gourtines之间用来设置截止日期、同步信号，传递请求相关值的结构体。

比如一个网络请求Request，每个Request都需要开启一个goroutine做一些事情，这些goroutine又可能会开启其他的goroutine。当请求超时时，需要关闭所有的goroutines：

```
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	go watch(ctx,"【监控1】")
	go watch(ctx,"【监控2】")
	go watch(ctx,"【监控3】")

	time.Sleep(10 * time.Second)
	fmt.Println("可以了，通知监控停止")

    // 主动关闭监控
	cancel()

	// 为了检测监控过是否停止，如果没有监控输出，就表示停止了
	time.Sleep(5 * time.Second)
}

// 在watch中必须使用select来监听Ctx的数据
func watch(ctx context.Context, name string) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println(name,"监控退出，停止了...")
			return
		default:
			fmt.Println(name,"goroutine监控中...")
			time.Sleep(2 * time.Second)
		}
	}
}
```

Context本身可以构建出一个树， 当父节点cancel时，其所有的子节点都被会cancel。

我们在实际项目中很少会用到，应该为实际项目中，大部分场景只会有一个parent，那就是`context.Background()`, 这个parent只会创建一个child，那就是WithCancel，或者是WithTimeout或者是WithDeadline之类的Context，然后这个Context会在goroutines之间传递。

在阅读context源码是，一定要建立Context树的概念，否则很难读懂代码。


## WithTimeout的使用
我们创建了一个过期时间为 1s 的上下文，并向上下文传入 handle 函数，该方法会使用 500ms 的时间处理传入的请求：
```
func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
	defer cancel()

	go handle(ctx, 500*time.Millisecond)
	select {
	case <-ctx.Done():
		fmt.Println("main", ctx.Err())
	}
}

func handle(ctx context.Context, duration time.Duration) {
	select {
	case <-ctx.Done():
		fmt.Println("handle", ctx.Err())
	case <-time.After(duration):
		fmt.Println("process request with", duration)
	}
}
```


## Deadline Context

在web服务器中，经常会出现一个错误:Context Deadline Exceeded。

这是因为，当HTTP请求的上下文有一个截止时间或超时设置，即请求应该在什么时间之后中止，在本次请求中，如果发现请求了RPC, Redis, Mysql之类的大概率是因为这些涉及到网络IO的服务访问超时导致的。

解决方法：
1. 判断是否访问RPC/Redis/Mysql导致的，减少配置时间和重试次数
2. 适当调整整个请求的超时时间

DeadLine Context使用：
```
package main

import (
	"context"
	"fmt"
	"time"
)


func main() {
    // 从配置中获取超时时间
    duration := GetFromConf()
	d := time.Now().Add(duration)

    // ctxt会一直传递下去, 发起RPC请求时，Rpc Client会判断是否超时
	ctx, cancel := context.WithDeadline(context.Background(), d)

	defer cancel()
	select {
	case <-time.After(1 * time.Second):
		fmt.Println("overslept")
	case <-ctx.Done():
		fmt.Println(ctx.Err())
	}
}

func RpcClient(ctx context.Context) {
    // 监听Rpc请求是否超时
    for {
        select {
            case <-ctx.Done():
                fmt.Println("handle", ctx.Err())
        }
    }
}
```
