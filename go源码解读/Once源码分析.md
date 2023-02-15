# Once

`Once` 保证函数只会执行一次, Demo:

```
package main

import (
	"fmt"
	"sync"
)

func main() {
	var once sync.Once
	onceBody := func() {
		fmt.Println("Only once")
	}

	done := make(chan bool)
	for i := 0; i < 10; i++ {
		go func() {
			once.Do(onceBody)
			done <- true
		}()
	}

	for i := 0; i < 10; i++ {
		<-done
	}
}

// 输出一次Only once
```

## 使用案例

1. 用于只输出一次错误日志

```
	for i := 0; i < n; i++ {
		go func() {
			if err := repo.Set(configs); err != nil {
				// Don't print 1000 errors, only the first.
				once.Do(func() {
					t.Errorf("concurrent connections to sqlite3: %v", err)
				})
			}
			wg.Done()
		}()
	}
```

2. 用于close channel, channel只能被close一次

```
	inboundEmpty := make(chan struct{})
	var onUpdateOnce sync.Once
	onUpdate := func(p *Peer) {
		if inbound, _ := p.NumConnections(); inbound == 0 {
			// 只会调用close一次
			onUpdateOnce.Do(func() {
				close(inboundEmpty)
			})
		}
	}
```

## 源码分析

```
type Once struct {
	// done等于1表示f函数已经执行完成
	done uint32

	// 使用互斥锁控制只有一个协程执行f函数
	m    Mutex
}

func (o *Once) Do(f func()) {
	// 这里有个不正确的实现方法:
	//
	//	if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
	//		f()
	//	}
	//
	// Do函数需要保证当它返回是，f函数已经执行完成
	// 但是这种实现方式无法保证Do返回时，f()已经执行完成
	// 假如有两个协程同时执行CAS，执行成功的协程接着执行f()
	// 执行CAS失败的协程立即返回，但此时f()有可能没有执行完成。

	// done等于0表示f未执行完成
	if atomic.LoadUint32(&o.done) == 0 {
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	// 加锁
	// 竞争锁失败的协程会处于阻塞状态
	o.m.Lock()

	// 唤醒其他阻塞的协程
	defer o.m.Unlock()

	// 再次判断done的值
	if o.done == 0 {
		// 通过defer到达1个效果：只有等f执行完成后才修改done的状态
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}

```

## 总结
1. 通过`Done`的两次检查实现Do方法，我们可以`double-check`思想
2. 两次调用 sync.Once.Do方法， 传入不同的函数，也只会执行第一次调用的函数
3. 执行的函数没有返回值
