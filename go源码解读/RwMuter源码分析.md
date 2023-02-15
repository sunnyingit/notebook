RWMutex 读写锁的实现

## 为什么需要使用【读写锁】

在大多数服务中，并发写会把数据写乱，而并发读则不会，同时读请求之间不会相互影响，无须担心读请求的并发。

同时读请求的比例远远大于写请求，支持【并发读】会大大提升服务性能。

因此，我们需要控制并发写，无须控制并发读，实现这类需求就会用到【读写锁】。

【读写锁】相对于【互斥锁】而言，【读写锁】是一把粒度更细的锁，在读场景多的情况下，效果会更好。

> 这种【读写分离】的思想，在数据库设计方面也经常被应用到。设计主从数据库，主数据库用来写数据，从数据库用来读。读写数据库的锁级别可以设置不一样。x

## 使用案例
```
type Stat struct {
    // 模拟资源，go的map类型是不允许并发读写的
	counters map[string]int64

	// 读写锁
	mutex    sync.RWMutex
}

func InitStat() Stat {
	return Stat{counters: make(map[string]int64)}
}

func (s *Stat) Count(name string) int64 {
	// 写锁
	s.mutex.Lock()
	defer s.mutex.Unlock()

	s.counters[name]++

	return s.counters[name]
}

func (s *Stat) GetCounter(name string) int64 {
	// 读锁
	s.mutex.RLock()
	defer s.mutex.RUnlock()

	return s.counters[name]
}
```

## 实现构想

首先，我们拆解一下【读锁】和【写锁】的需求：

对【读锁】而言：

1. 加【读锁】，需要判断是否有协程加【写锁】成功，如果已经尝试加了写锁(即使写锁没有加成功，则后续不可以再加读锁，尝试加的协程都被阻塞)。
2. 解【读锁】，只有当所有的【读锁】都释放后，才判断是否有【写协程】等待【读锁】而阻塞，有则唤醒。

对【写锁】而言：
1. 加【写锁】，写写操作之间互斥，需要优先判断有没有其他协程加【写锁】成功，没有则成功，有则阻塞
2. 加【写锁】，拿到互斥执行权后，需要判断是否有其他协程加了【读锁】，没有则成功，有则阻塞，直到等待所有的【读锁】释放。
3. 解【写锁】，判断是否有【读协程】等待【写锁】释放，有则唤醒所有的【读协程】。
4. 解【写锁】，释放其他等待【互斥锁】的协程。

更细致的需求拆解：
1. 需要用一个变量保存“是否有协程加写锁成功”。
2. 需要用一个变量保存“已经有多少协程加读锁成功”。
3. 需要用一个变量保存“写锁在等多少个读锁释放”。
4. 利用【互斥锁】来实现写写之间互斥。
5. 读，写信号量

为啥没有“读锁在等多少个写锁释放”？
因为只要有一个协程加写锁成功，所有的协程加读锁都失败，所以不需要记录“读锁在等多少个写锁释放”

看看go的源码的实现是否符合我们的预期。

## 读锁源码

```
type RWMutex struct {
    // w是一把互斥锁，用于控制【写写】操作之间的并发
	w  Mutex

	// 写信号
	writerSem   uint32

    // 读信号
    readerSem   uint32

    // 成功加【读锁】的协程的数量
	readerCount int32

	// 如果协程A加【写锁】时，如果此时已经有R个协程加【读锁】成功
    // 则协程A需要等待R个写锁释放【读锁】后，才能加【写锁】成功
    // readerWait就是表示等待多少个协程释放【读锁】
	readerWait  int32
}

// 加读锁
func (rw *RWMutex) RLock() {

    // 调用原子操作AddInt32函数，将加【读锁】的协程数加1
    // 然后在判断加1之后是不是小于0，小于这陷入阻塞
    // 这里通过readerCount 是一个超大的负数来模拟，是否有协程加【写锁】成功。
    // 并非我们预期使用一个变量来表示“是否有协程加【写锁】成功”
	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
		runtime_SemacquireMutex(&rw.readerSem, false, 0)
	}
}

// 解【读锁】
func (rw *RWMutex) RUnlock() {
    // 调用原子操作AddInt32函数，将加【读锁】的协程数减1
    // 减1之后，如果还是【负数】，说明此时有协程因为执行【写锁】, 改变了readerCount的值，
    // 当r为负数时，表示已经有写锁阻塞了，需要通知释放写锁。
	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        // r表示目前已经有多少个读锁，只有这些读锁都释放后，才可以继续加写锁或者读锁
		rw.rUnlockSlow(r)
	}
}


func (rw *RWMutex) rUnlockSlow(r int32) {
    // r+1 == 0 ，那就是r=-1, 那就说明调用RUnlock时，readerCount=0, 也就是说明没有执行【RLock】。
    // 也就是没有加【读锁】，但去释放【读锁】
    // r+1 == -rwmutexMaxReaders 表示 加了【写锁】，而释放【读锁】。
	if r+1 == 0 || r+1 == -rwmutexMaxReaders {
		throw("sync: RUnlock of unlocked RWMutex")
	}

    // 如果协程A加【写锁】，但是已经有readerWait协程已经获取【读锁】。
    // 每次释放读锁， 则readerWait都减1， 说明协程A等待的数量减1
    // 当readerWait减少到1的时候，就可以唤醒协程A去加【写锁】了
    // 这就是atomic.AddInt32(&rw.readerWait, -1) == 0的意思
	if atomic.AddInt32(&rw.readerWait, -1) == 0 {
		runtime_Semrelease(&rw.writerSem, false, 1)
	}
}

```

## 写锁源码
```
func (rw *RWMutex) Lock() {
    // 先加互斥锁，保证同时只能有一个协程加【写锁】成功。
	rw.w.Lock()

    // 把readerCount设置为一个负数, 用来表示，此时有协程尝试加【写】锁，此时无法再加读锁
    // 只有已经加的读锁都释放了，才可以继续加其他锁。
	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders

	// r!=0，说明已经有协程加【读锁】成功，需要等待这些读锁释放之后，【写锁】才能成功
    // 调用 AddInt32 把 readerWait 加上r, 表示需要等待r个协程释放【读锁】
	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
		runtime_SemacquireMutex(&rw.writerSem, false, 0)
	}
}

// 解【写锁】
func (rw *RWMutex) Unlock() {
    // 先把readerCount变成正数，此后，加【读锁】可以成功
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)

    // r >= rwmutexMaxReaders 表示readerCount >= 0，表示加了【读锁】，而解【写锁】
	if r >= rwmutexMaxReaders {
		throw("sync: Unlock of unlocked RWMutex")
	}

	// 唤醒所有的【读锁】
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false, 0)
	}

	// 唤醒其他加【读锁】而阻塞的协程
	rw.w.Unlock()
}
```

## 思考

请看下面代码
```
func (s *Stat) GetCounter(name string) int64 {
	// 读锁
	s.mutex.RLock()
	defer s.mutex.RUnlock()

	return s.counters[name]
}
```
试想，有没有一种情况，协程A，协程B同时加读锁，因为`AddInt32`的原子性保证，只会有一个协程执行成功。假如协程A加读锁成功后，还没有来得及执行`s.counters[name]`， 协程A所在的线程因为时间片到期被剥夺了CPU执行权，之后协程B在也加读锁成功，如果协程A被唤醒，那有没有可能协程A和协程B同时访问s.counters呢？

答案是不会，协程A会在线程交出CPU控制权前会把该协程移动到其他线程上。

## 设计启发
1. 读写分离思想。
2. 用“正负关系”来表示两种状态。
3. 处理“加锁和解锁不成对”的异常。
