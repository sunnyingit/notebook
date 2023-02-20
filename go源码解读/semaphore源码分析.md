# semaphore

信号量主要用于控制并发数，常见使用场景包括：
1. 限流，控制并发请求的数
2. 秒杀，保证只有指定数量的商品能被抢购
3. 工作池，保证同时只有指定数量的协程在工作


信号量使用逻辑是：
1. 初始化信号量时，指定计数器的最大值
2. 获取资源时，将信号量中的计数器减去对应的值, 当申请的值大于计数器剩下的值, 则陷入阻塞
3. 释放资源时，将信号量的计数器加上对应的值, 同时尝试去唤醒所有阻塞的协程

`semaphore.NewWeighted` 暴露四个方法用于控制信号量：
1. `NewWeighted` 初始化信号量
2. `Acquire` 获取资源
3. `TryAcquire` 无阻塞式的获取资源，成功返回true，否则返回false
4. `Release` 释放资源

请看demo, 演示"只允许10个协程同时执行"：

、、、

package main

import (
	"context"
	"log"
	"time"

	"golang.org/x/sync/semaphore"
)

func RPC(id int) {
	// monitor rpc request...
	time.Sleep(1 * time.Second)
}

func main() {
	// 初始化信号量, 指定
    var sem = semaphore.NewWeighted(int64(10))
	maxWorkers := 10
	pool := make([]int, maxWorkers)

	// 初始化一个ctx， 可以在外部调用ctx.Done终止协程
	ctx := context.Background()
	for id := range pool {
		// 当信号量足够时，Acquire执行成功，不足时则协程阻塞， 如果返回错误，则有两种可能：
		// 1.外部协程调用ctx.Done
		// 2.申请的信号量大于总量
		if err := sem.Acquire(ctx, 1); err != nil {
			log.Printf("Failed to acquire semaphore: %v", err)
			break
		}

		// 开启worker协程
		go func(id int) {
			// 不要在新协程内调用Acquire，最好在此协程执行前调用Acquire
			// 因为main协程中调用了sem.Acquire(ctx, int64(maxWorkers))
			// 有可能main协程先执行Acquire，但main协程没有调用release, 这样新协程执行Acquire失败,程序陷入死锁
			// sem.Acquire(ctx, 1)
			// 建议使用defer执行release防止调用RPC异常后，没有执行release导致死锁
			defer sem.Release(1)
			RPC(id)
		}(id)
	}

	// 阻塞直到所有的worker都调用release
	if err := sem.Acquire(ctx, int64(maxWorkers)); err != nil {
		log.Printf("Failed to acquire semaphore: %v", err)
	}

	// 做一些业务逻辑。。。。
	// 程序执行到这里说明worker都执行完成
	sem.Release(int64(maxWorkers))
}

、、、


在秒杀的场景中，我们可以用商品的总量初始化`NewWeighted`, 先使用`TryAcquire`模拟是否拿到抢购资格，
`TryAcquire`返回`true`表示拿到资格，然后执行购买商品逻辑，若购买过程出现异常，则调用`Release`释放资格。
如果`TryAcquire`返回`false`则表示商品已经抢完。


在限流场景中，我们同样可以调用`TryAcquire`模拟是否拿到执行权。


## 设计思路

老规矩，我们来尝试分析下`semaphore`设计思路，根据我们对`semaphore`的理解，可以整理出需求如下：
1. 同时有多个协程调用`Acquire`, `Release`, 保证只能有一个协程执行成功
2. 需要两个计数器，一个用于保存信号总量，一个用于保存已使用信号量
3. 需要一个数据结构保存等待协程的列表
4. 需要考虑如何阻塞协程，如何唤醒协程
5. 需要考虑清楚协程阻塞的条件，被唤醒的条件

下面我们看看源码是如何实现这些需求。

## 源码分析

// 模拟信号量
type Weighted struct {
	// 保存信号量的总数
	size    int64

	// 保存已使用的信号量数，剩余的信号量=size-cur
	cur     int64

	// 互斥锁，用于保证Acquire, Release的互斥性
	mu      sync.Mutex

	// list用于保存所有的waiter
	waiters list.List
}

// 初始化信号量
func NewWeighted(n int64) *Weighted {
	w := &Weighted{size: n}
	return w
}

// waiter模拟“等待的协程”，让协程阻塞在ready管道上
// 当可用的信号量大于或者等于n时，调用close(ready)来唤醒协程
type waiter struct {
	n     int64
	ready chan<- struct{}
}


// 获取信号量
func (s *Weighted) Acquire(ctx context.Context, n int64) error {
	// 保证互斥性
	s.mu.Lock()

	// 当剩余信号量大于申请的信号量n, 而且没有等待的waiters，才申请成功
	// 为什么需要判断s.waiters.Len() == 0呢
	// 这是为了保证公平性，让先申请的协程优先得到资源，注意此时，先申请的协程可能处于阻塞态了
	// 如果waters不为空，则优先考虑唤醒阻塞的waiter，而不是满足执行Acquire的协程
	// 不得不说，实现很巧妙
	if s.size-s.cur >= n && s.waiters.Len() == 0 {
		// 已分配的长度+n
		s.cur += n
		// 解锁
		s.mu.Unlock()
		return nil
	}

	// 如果申请的信号量大于初始化的总量，则直接返回错误
	if n > s.size {
		s.mu.Unlock()
		<-ctx.Done()
		return ctx.Err()
	}

	// 程序执行到这里，此协程需要阻塞直到其他协程调用release
	ready := make(chan struct{})

	// 初始化waiter, n表示唤醒此协程需要的可用信号量
	w := waiter{n: n, ready: ready}

	// 加入到链表中
	elem := s.waiters.PushBack(w)

	// 解锁，注意一定要在执行select之前，否则会死锁
	s.mu.Unlock()

	// select一直阻塞，直到：
	// 1. 是否外部协程调用了ctx.cancel, 如果是, 则走到cxt.Done逻辑
	// 2. 其他协程调用了release,走到ready逻辑, 直接返回nil
	select {
	case <-ctx.Done():
		err := ctx.Err()
		s.mu.Lock()
		select {
		case <-ready:
			// 如果此时有些协程调用了release, 则直接返回
			err = nil
		default:
			// 需要手动释放waiters
			isFront := s.waiters.Front() == elem
			s.waiters.Remove(elem)
			if isFront && s.size > s.cur {
				s.notifyWaiters()
			}
		}

		// 解锁
		s.mu.Unlock()
		return err

	case <-ready:
		return nil
	}
}

// 释放信号量
func (s *Weighted) Release(n int64) {
	// 保证互斥性
	s.mu.Lock()

	// 已使用的信号量减n
	s.cur -= n

	// 如果cur为负数，说明Release的信号量大于已使用的信号量
	if s.cur < 0 {
		s.mu.Unlock()
		panic("semaphore: released more than held")
	}

	// 唤醒waiters
	s.notifyWaiters()

	// 解锁
	s.mu.Unlock()
}



// 唤醒所有waiter
func (s *Weighted) notifyWaiters() {
	// 遍历waiters链表
	for {
		next := s.waiters.Front()

		// next=nil表示链表遍历完成
		if next == nil {
			break
		}

		// 类型转换成waiter
		w := next.Value.(waiter)

		// 如果剩余的信号量不满足阻塞协程需要的数，直接让waiter继续阻塞
		// 等待其他的协程调用release释放出足够的信号量
		// 这里直接使用break退出程序，而不是继续遍历其他waiter。
		// 这样做是为了防止waiter一直无法释放而饿死。
		if s.size-s.cur < w.n {
			break
		}

		// 程序执行到这里，说明可以此waiter，所以使用的信号量需要加n
		s.cur += w.n

		// 移除waiter
		s.waiters.Remove(next)

		// 调用close唤醒waiter, waiter的select会接受到信号
		close(w.ready)
	}
}

// 无阻塞的获取资源
func (s *Weighted) TryAcquire(n int64) bool {
	s.mu.Lock()
	success := s.size-s.cur >= n && s.waiters.Len() == 0
	if success {
		s.cur += n
	}
	s.mu.Unlock()
	return success
}

## 总结
信号量主要用于控制并发数, 再次提醒下，信号量可以用于【限流】，【秒杀】，【工作池】等场景，在使用的过程中，需要注意几个问题：

1. 需要考虑获取资源时是否需要阻塞。
2. Acquire的信号量不能超过总数限制，Release释放的信号量不得超过已使用的信号量
3. 通过FIFO的算法来唤醒阻塞的协程，保证公平性
4. semaphore使用waiter结构体来实现阻塞，唤醒逻辑非常巧妙


最后，说一个小技巧，`semaphore`还可以模拟读写锁：

、、、
// 加读锁
Acquire(1)

// 释放读锁
Release(1)

// 加写锁, N表示信号总量
Acquire(N)

// 释放写锁
Release(N)
、、、
