阅读本文可以获得：
1. 理解锁的基础概念
2. 理解锁的实现原理
3. 理解锁对性能的影响

在并发编程中，离不开锁概念，锁保证了在并发访问【临界资源】时，不会出现【竞争 Race condition】。我们将以`Sync.Mutex`源码为例，分析互斥锁的实现原理。

Go的`Sync.Mutex`包提供了`Lock`和`Unlock`两个函数，分别用于加锁和解锁，使用非常简单：

```
func main() {

	source := map[string]string{
		"key": "value",
	}

	// 模拟两个协程同时访问source
	var r sync.Mutex
	go func() {
		r.Lock()
		fmt.Println(source["key"])
		r.Unlock()
	}()

    // 模拟两个协程同时访问source
	go func() {
		r.Lock()
		fmt.Println(source["key"])
		r.Unlock()
	}()

	// main协程等着
	for {
	}
}

```

分析互斥锁原理之前，我们需要了解和锁的相关概念


## 阻塞
阻塞是线程的一种状态，线程让出CPU的执行权，当线程满足可执行状态后，能再次被操作系统调度，占用CPU的控制权，此时线程处于运行态。线程状态的变化过程，称为上下文调度，此过程需要消耗时间和内存。
当线程处于阻塞态时，可以简单等同于，线程上patch的goroutine也处于阻塞态。下文直接表述为goroutine处于阻塞态。


## Spin
spin 翻译为自旋，此时CPU会循环执行空指令，自旋避免了goroutine进入阻塞状态，节省了上下文的调度开销，弊端是自旋时间过长会造成CPU浪费。

## CAS
CAS是一种原子操作，该操作通过将内存中的值与指定数据进行比较，当数值一样时将内存中的数据替换为新的值。当多个线程同时执行CAS时，只会有个线程执行成功。
CAS的原子性保证是硬件CPU支持的，而非操作系统。目前，大部分CPU都支持CAS操作。


## 并发事实
程序并发访问意味着，同时有多个goroutine访问【临界资源】，当我们调用`Lock`&`Unlock`控制并发访问【临界资源】时，需要保证：
1. `Lock`和`Unlock`，同时只会有一个操作成功。
2. 可能有多个goroutine调用`Lock`加锁，但只会有一个goroutine执行成功。
3. 可能有多个goroutine调用`Unlock`加锁，但只会有一个goroutine执行成功。
4. 加锁失败的goroutine会处于阻塞或者spin状态, 但不能一直处于Spin状态。
5. 解锁后需要自动唤醒goroutine去获取锁，此时，这类goroutine会与调用`Lock`的goroutine产生竞争，需要做到这两类goroutine都有机会拿到锁。


## 锁的实现

根据并发的事实，我们实现`Lock`和`Unlock`的需求是：
1. 锁的数据结构设计，能够反应出`加锁`和`解锁`两种状态。
2. 加锁失败的goroutine有可能进入`spin`状态，`spin`结束后再进入`阻塞`状态。
3. 设计一个队列，保存所有因为无法获取到锁而阻塞的goroutine。
4. 设计通知机制，当解锁之后，能通知到阻塞的goroutine尝试去获取锁。
5. 保证公平性：当被唤醒的goroutine和调用`Lock`函数的goroutine同时竞争，往往调用`Lock`函数的goroutine会拿到锁，因为调用`Lock`函数的goroutine的数量比较多，而且他们正占有着CPU的控制权。被唤醒的goroutine因拿不到锁而再次阻塞，这样会导致阻塞goroutine会长时间拿不到锁而`饿死`。


Go的`runtime`包自带函数`runtime_Semrelease`, `runtime_SemacquireMutex` 已经帮我们实现了需求【3-4】。
执行`runtime_SemacquireMutex`会导致当前goroutine阻塞，当其他goroutine调用`runtime_Semrelease`时会通知被阻塞的goroutine继续执行。 这两个函数不是本文重点，我们只需要了解大概意思。

下面将分享三种版本【简洁版】【升级版】和【终极版】。这三版都是Go不同版本的源码。


### 简洁版
这一版中，我们只需要实现需求【1, 3, 4】。

Go [实现](https://github.com/golang/go/commit/dd2074c82acda9b50896bf29569ba290a0d13b03?branch=dd2074c82acda9b50896bf29569ba290a0d13b03&diff=split) 如下：

```
type Mutex struct {
	// key的值表示锁的状态 0: 未加锁 1: 加锁
	key  int32
	sema uint32
}

func (m *Mutex) Lock() {
	// 使用原子操作AddInt32, 同时只有一个goroutine执行此函数成功, 同时保证了Lock, Unlock只有一个执行成功。
	// 加1之后还是1的话，说明原状态是0，表示获取到锁
	if atomic.AddInt32(&m.key, 1) == 1 {
		return
	}

	// 未获取到锁，则直接阻塞
	runtime.Semacquire(&m.sema)
}

func (m *Mutex) Unlock() {
	// 使用原子操作AddInt32, 同时只有一个goroutine执行此函数成功，同时保证了Lock, Unlock只有一个执行成功。
	// 减1之后如果v为0，则说明解锁成功，否则抛出异常
	switch v := atomic.AddInt32(&m.key, -1) {
	case v == 0:
		return
	case v == -1:
		panic("sync: unlock of unlocked mutex")
	}

	// 通知阻塞的协程尝试去获取锁
	runtime.Semrelease(&m.sema)
}
```

此版本存在的问题是，一旦加锁失败就陷入阻塞状态，而此时有可能其他goroutine释放了锁，理论上如果此时再尝试去获取锁时会成功，也就是无须陷入阻塞状态。

如果改进这个问题呢？


### 升级版

请看升级版代码：

```
type Mutex struct {
    // 锁的状态： 0：未加锁， 1：加锁，2：有woken的协程在尝试获取锁
    // state右移mutexWaiterShift表示阻塞协程的数量
	state int32
	sema  uint32
}

const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken
	mutexWaiterShift = iota
)

// 加锁
func (m *Mutex) Lock() {
    // 使用CAS尝试去获取锁，如果m.state值是0，则获取锁成功，同时设置锁的状态为mutexLocked
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		return
	}

    // 如果获取锁失败，需要改变锁的状态

    // 主动调用Lock加锁的协程awoke设置为false
	awoke := false

	for {
        // 锁的旧状态，值得注意的是，此时锁的状态不一定是mutexLocked状态
        // 也许你会问，既然锁的状态不是mutexLocked，为何执行CompareAndSwapInt32失败。
        // 因为此时，有可能其他协程已经解了锁，所以m.state可能是未加锁状态。
		old := m.state

        // 设置锁的新状态
        // 也许你会问，既然有可能锁是为加锁状态，为什么这里还要设置mutexLocked，而且此协程还没有拿到锁。
        // 答案这里new状态是有可能的状态，不一定会设置成功
		new := old | mutexLocked

        // 如果此时锁还处于已加锁状态，阻塞协程的数量加1
		if old&mutexLocked != 0 {
			new = old + 1<<mutexWaiterShift
		}

        // 如果这段for程序是被唤醒协程执行，那此时awoke为true
        // 如果再次获取锁失败时，更新锁的状态：没有awoken的协程获取锁
		if awoke {
			new &^= mutexWoken
		}

        // 更新锁的状态，注意加锁失败的协程都在更新锁的状态，但只有一个更新成功
        // 如果不成功，则会继续执行for循序，再次尝试
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
            // 如果CAS更新成功。
            // 再次判断锁的状态，如果是解锁状态，那说明此时锁已经被其他线程解锁了
            // 那上面old&mutexLocked != 0执行必定为false, 那就是没有执行 "阻塞协程的数量加1"操作
            // 锁新状态是new := old | mutexLocked
			if old&mutexLocked == 0 {
			    // 直接返回，说明拿到了“锁”，并且设置了锁的状态是mutexLocked
				break
			}

            // 程序走到这里说明加锁失败，那old&mutexLocked != 0执行必定成功。也就是执行了“阻塞协程的数量加1”
            // 执行Semacquire, 此时goroutine才陷入阻塞状态
			runtime.Semacquire(&m.sema)

            // 当程序执行到这里时，说明goroutine被唤醒了

            // 此时设置awoke, 此goroutine会再次执行for循环，尝试去改变锁的状态。
			awoke = true
		}
	}
}

// 解锁
func (m *Mutex) Unlock() {

    // 去掉“mutexLocked”状态后，此时new并不等于0，因为有可能阻塞的协程数量不为0.
    // 再次注意，state不仅表示锁的状态，还表示阻塞协程的数量
	new := atomic.AddInt32(&m.state, -mutexLocked)

    // 这里防止只调用了一次Lock，但多次调用Unlock，抛出异常。
	if (new+mutexLocked)&mutexLocked == 0 {
		panic("sync: unlock of unlocked mutex")
	}

	old := new
	for {

        // old>>mutexWaiterShift == 0 表示并没有阻塞的协程了，则直接返回。
        // ld&(mutexLocked|mutexWoken) !=0 只有一种情况，那就是被唤醒的协程正在准备去获取锁，所以直接返回。
		if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken) != 0 {
			return
		}

        //设置“阻塞协程的数量减1”， 设置锁的状态为mutexWoken
		new = (old - 1<<mutexWaiterShift) | mutexWoken

        // 同时有多个协程在调用Unlock, 只会有一个执行成功
		if atomic.CompareAndSwapInt32(&m.state, old, new) {

            // 执行成功后通知阻塞的协程去获取
            runtime.Semrelease(&m.sema)
			return
		}

        // 如果CAS执行不成功的话，更新old的值，再次尝试执行
		old = m.state
	}
}

```

这一版对比【简洁版】主要的升级是真正引入锁的状态，而且还记录了阻塞协程数量，可以尽量减少`Semrelease`, `Semacquire` 这类【系统调用】操作，同时最大程度避免了【上下文开销】。


### 终极版
到目前为止，我们还剩一个需求没有实现，那就是【加锁失败的goroutine有可能进入`spin`状态，`spin`结束后再进入`阻塞`状态】。

下面，我们来看下Go终极实现，也就是现在`Sync.Mutex`的实现：

```
type Mutex struct {
	state int32
	sema  uint32
}

const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken

    // 增加了一个饥饿状态
	mutexStarving
	mutexWaiterShift = iota

    // 超过1ms就会处于starvation状态
	starvationThresholdNs = 1e6
)

func (m *Mutex) Lock() {
    // 使用CAS尝试去获取锁，如果m.state值是0，则获取锁成功，同时设置锁的状态为mutexLocked
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		return
	}

	// 如果CAS执行失败，则往下继续执行
	m.lockSlow()
}

func (m *Mutex) lockSlow() {
    // 记录当前goroutine开始block的时间
	var waitStartTime int64

    // 记录当前的goroutine是否处于starving状态
	starving := false

    // 记录当前的goroutine是否处于awoke状态
	awoke := false

    // 记录当前的goroutine spin的计数
	iter := 0

	// 获取当前锁的状态
	old := m.state
	for {
        // 判断是否需要spin， 进入spin的条件为：
        // 1. 锁是mutexLocked状态且不是`mutexStarving`, 如果处于`mutexStarving`则锁的控制权会给到weaken的goroutine
        // 2. runtime_canSpin 返回为true, 此函数返回为true的条件是：
        // 2.1 运行在多 CPU 的机器上
        // 2.2 当前 Goroutine 为了获取该锁进入自旋的次数小于四次
        // 2.3 当前机器上至少存在一个正在运行的处理器P并且处理的运行队列为空
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // old>>mutexWaiterShift != 0 表示有阻塞的协程在等这把锁, 只有这种情况下才会设置woken
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 && atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}

			// 开始spin, spin时当前goroutine会占有CPU控制，持续执行"无效"指令
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}

        // spin结束后此时，锁的状态可能有多种：
        // 1. 解锁且处于starvation状态
        // 2. 解锁且不处于starvation状态
        // 3. 未解锁且处于starvation状态
        // 4. 未解锁且不处于starvation状态

        // 当前协程尝试去修改锁的状态
		new := old

        // 如果不处于mutexStarving, 则就会尝试去加锁，锁的新状态为mutexLocked
        // 如果锁处于mutexStarving状态，则什么都不干，因为锁的控制权只会给到阻塞的goroutine
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}

        // 如果锁处于mutexLocked或者mutexStarving, 则尝试去设置锁的等待的goroutine数+1
        // new左移mutexWaiterShift位后拿到的就是阻塞的goroutine的数量
        // 别担心把阻塞的数量加多了，万一此goroutine拿到锁之后, 会把这个数量减1的，这种思路，我称之为"补偿"编程。
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}

        // 如果此goroutine等待时间超1ms且锁是mutexLocked状态，则设置该锁处于mutexStarving状态
        // 如果锁状态不是mutexLocked，此goroutine可能拿到锁，也就没有必要设置锁状态为mutexStarving
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}

        // 如果此goroutine是被唤醒后去竞争锁的，则去掉锁的mutexWoken状态
		if awoke {
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}

        // 调用CAS尝试去修改锁的状态, 同时只会有一个幸运goroutine会执行CAS成功, 失败的goroutine会再次尝试修改锁状态，直到成功。
        // 是否会有goroutine很长时间都成功无法修改锁状态，导致加锁超时？理论上是存在的。
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
            // 重点：如果修改锁状态成功后，如果锁【old状态】不处于mutexLocked|mutexStarving, 说明锁已经被其他goroutine解锁了
            // 回头去看，在修改锁状态的代码中，只可能有old&mutexStarving == 0判断为true, 并设置 new |= mutexLocked
		    // 通过CompareAndSwapInt32设置m.state=new，即锁的状态重新设置为mutexLocked状态
            // 最后通过break直接退出程序。
			if old&(mutexLocked|mutexStarving) == 0 {
				break
			}

            // 执行到这里，说明锁还是处于mutexLocked或者mutexStarving状态, 此goroutine需要开始blocked了
            // blocked之前记录一下blocked的开始时间waitStartTime
            // 如果waitStartTime不等于0，说明此goroutine是从被唤醒后去竞争锁的，走到这里，说明竞争锁还是失败了
            // 这种情况下，把此goroutine放到阻塞队列的对头，下次将第一个出来竞争，此时queueLifo=true
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}

            // 把此goroutine放到阻塞队列的对头, 并阻塞当前的goroutine，剥夺CPU的执行权，可怕的睡眠开始了。。。。。
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)

            // 下面的代码开始执行，说明只会有一种情况: 有其它的goroutine调用了Unlock，并唤醒阻塞的goroutine再次去竞争锁

            // 竞争之前，判断此goroutine是不是等到快饿死了
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs

            // 获取锁的状态
			old = m.state

            // 如果锁还没有处于starvation状态，则走到else代码中，程序重新从for开始执行，尝试去获取锁
            // for执行第一轮的时候starving初始化为false, 第二轮之后starving的值就可能会true
            // 如果锁已经是starvation状态
			if old&mutexStarving != 0 {
                // 程序走到这里，说明协程快饿死，终于能拿到锁了，喜极而泣.....

                // 如果处于mutexStarving, 最起码可以推导处于三个结论：
                // 1. 阻塞的goroutine数量至少有一个，就是当前的goroutine
                // 2. 锁不可能处于mutexLocked，为啥，因为此goroutine代码执行到这里，说明被Unlock唤醒, 且如果锁已经处于mutexStarving
                // 那就不可能有其他goroutine能成功设置锁的状态为mutexLocked。
                // 3. 不可能处于mutexWoken，可回头看这行代码:new &^= mutexWoken
                // 不满足这三点就会抛出异常
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}

                // 设置锁的状态为mutexLocked, 并且blocked的gourintes数量减1
				delta := int32(mutexLocked - 1<<mutexWaiterShift)

                // 如果此时阻塞队列只有一个goroutine, 也就是说当前的goroutine是最后一个，那就直接去掉锁的mutexStarving
                // 如果此goroutine等待的时间不超过1ms，那就去掉锁的mutexStarving状态
				if !starving || old>>mutexWaiterShift == 1 {
					delta -= mutexStarving
				}

                // AddInt32结合上面的几行, 可以翻译为：
                // 加锁：m.state= m.state + mutexLocked
                // goroutine数量减1：m.state = m.state - (1<<mutexWaiterShift)
                // 取消starvation状态 m.stat = m.state  - mutexStarving
				atomic.AddInt32(&m.state, delta)
				break
			}

            // 如果还是没有拿到锁，则设置goroutine为woken状态
			awoke = true

            // 唤醒后的goroutine会重新开始spin计数
			iter = 0
		} else {
			old = m.state
		}
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}

// 解锁
func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}

    // 去掉锁mutexLocked的状态
	new := atomic.AddInt32(&m.state, -mutexLocked)

    // 去掉之后，new不等于0，说明锁有阻塞的goroutine或者处于starvation状态
	if new != 0 {
		m.unlockSlow(new)
	}
}

func (m *Mutex) unlockSlow(new int32) {
    // 不可以多次调用Unlock否则异常
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}

    // 如果锁不处于starvation状态
	if new&mutexStarving == 0 {
		old := new
		for {
            // old>>mutexWaiterShift == 0 表示如果阻塞的goroutine数量为0，就不需要状态通知了
            // old&(mutexLocked) !=0 表示如果此时其他的goroutine又重新加锁成功，则通知的逻辑应该由
            // old&(mutexWoken) !=0 表示已经有被唤醒的goroutine在等这把锁，就无须通知其他goroutine
            // old&(mutexStarving) !=0 表示一定有个goroutine把锁的状态设置为mutexStarving, 并且在尝试获取锁
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}

            // 把阻塞的协程数量减1，设置锁状态为mutexWoken, 通知某个goroutine
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
                // 通知阻塞的协程尝试去获取锁
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
        // 处于starvation状态，则唤醒阻塞队列的头部的goroutine
		runtime_Semrelease(&m.sema, true, 1)
	}
}

```
【终极版】相对于【升级版】，主要是增加了协程的【spin】昨天，同时如果某个协程等锁等到快饿死的时候，能优先获取到锁，这个机制解决了锁的公平性问题。


## 源码启发
`Lock`和`Unlock`使用起来非常简单，但是实现却非常复杂，通过这三版go源码，我们可以很清晰的看到`Sync.Mutex`的优化，总结一下:
1. 通过位运算，巧妙的通过一个字段来表达多种状态。
2. 抽象出了锁的【mutexStarving】状态，解决了锁的【公平行】。
3. 实现了协程的【spin】状态，减少上下文开销，增加了锁的性能。
4. CAS的使用，一次操作二次调用CAS，一次用来【判断】锁的状态，一次用来【修改】锁的状态，这种思想非常值得我们借鉴。
5. 尝试修改锁状态时，使用【补偿性】操作，非常巧妙。
6. 理解【阻塞】【spin】,【CAS】，【公平性】等锁的相关概念以及实现
