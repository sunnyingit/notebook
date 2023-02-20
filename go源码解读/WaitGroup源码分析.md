
# WaitGroup

在并发场景中，我们需要协程之间协同合作，完成一件事情。常见的场景是批量发出HTTP请求, 等待所有的请求都完成后, 程序才能继续执行。

这种情况下，我们可以使用`WaitGroup`达到目的：

、、、
	var wg sync.WaitGroup

	for _, url := range urls {
		wg.Add(1)
		go func(url string) {
			defer wg.Done()
			f.Get(url)
		}(url)
	}

	// Wait阻塞，直到等待所有的url请求完成
	wg.Wait()
、、、

`Add`之后执行的协程，我称之为【先行】协程，`Wait`之后执行的协程，我称之为【等待】协程。


如果我们尝试去实现这种协作模式，必须实现如下逻辑：
1. 需要记录【先行】协程的个数，每个先行协程执行完成后，【先行】个数减1，如果个数减少到0，则唤醒所有等待的协程。
2. 需要记录【等待】协程的个数，这样可以唤醒所有等待的协程。
3. 需要一个信号量，用于阻塞和唤醒对应的协程。

我们来看下`sync.WaitGroup`是如何实现以上逻辑。

## 源码分析
`sync.WaitGroup` 提供了3个函数：
1. Add(delta int): 使【先行】协程的数量加delta，delta可能为负数。
2. Done(): 直接调用wg.Add(-1), 表示需要【先行】协程的个数减1。
3. Wait(): 阻塞当前协程，直到【协程】协程的个数减少到0时才被唤醒。


、、、
// 结构体
type WaitGroup struct {
	// sync.noCopy 是一个特殊的私有结构体
	// 保证了WaitGroup无法被copy
	noCopy noCopy

	// state1是长度为3的数组，分别用于记录【等待】个数,【先行】个数，信号量
	// 在64位机器中，state1[0]记录【等待】个数，state1[1]记录【先行】个数，state1[2]记录信号量。
	state1 [3]uint32
}


// 获取state信息
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
	// 用指针的对齐位是8位来判断是64位机器
	if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
		// 把两个uint32转换成一个uint64
		return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
	} else {
		return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
	}
}


// 修改【先行】协程的个数
func (wg *WaitGroup) Add(delta int) {
	statep, semap := wg.state()

	// uint64(delta)<<32 保证执行AddUint64的时候只会在高32位加上delta
	state := atomic.AddUint64(statep, uint64(delta)<<32)

	// 高32位是【先行】协程的个数
	v := int32(state >> 32)

	// uint32(state)取的是低32位，也就是【等待】协程个数
	w := uint32(state)

	// 【先行】的个数不可能为负数，发生这种只能是调用Done次数不对。
	if v < 0 {
		panic("sync: negative WaitGroup counter")
	}

	// 修改【先行】个数完成后，判断是否需要唤醒等待的协程:
	// v > 0 说明 还需要等【先行】协程执行完成，才能唤醒【等待】协程
	// w == 0 说明【等待】的协程个数为0，不需要唤醒。
	// 这两种情况都直接返回
	if v > 0 || w == 0 {
		return
	}

	// 程序执行到这里，说明【先行】协程已全部执行完成
	// statep=0 表示重置【先行】，【等待】的协程个数为0
	*statep = 0

	// 唤醒所有等待的协程
	for ; w != 0; w-- {
		runtime_Semrelease(semap, false, 0)
	}
}

// 减少【先行】个数
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}

// 等待直到被唤醒
func (wg *WaitGroup) Wait() {
	// 获取状态
	statep, semap := wg.state()

	for {
		// 使用原子操作读取指针的值
		state := atomic.LoadUint64(statep)

		// 【先行】的个数是高32位，所以需要右移
		v := int32(state >> 32)

		// uint32(state)取的是低32位，也就是【等待】个数
		w := uint32(state)

		// v==0，说明所有的【先行】协程都执行完成，可直接返回
		if v == 0 {
			return
		}

		// 使用CAS将【等待】协程的个数加1
		// 这里可能有多个协程同时调用Wait, CAS保证只有一个协程执行成功。
		// 失败的协程需要重新执行for循环，更新state后，在加1
		if atomic.CompareAndSwapUint64(statep, state, state+1) {

			// 加1之后，陷入阻塞，知道被唤醒
			runtime_Semacquire(semap)

			// 被唤醒后，说明【先行】协程的个数为0，同时执行了*statep = 0
			// 如果此时*statep不等于0，说明Wait在Add前被调用，这是不被允许的
			if *statep != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}

			return
		}
	}
}

、、、

注意，Go的`Happen-Before`原则，`Wait`方法一定会在`Add`方法之后执行。所以保证正常情况下不会出异常: `panic("sync: WaitGroup is reused before previous Wait has returned")`


## 总结
1. sync.WaitGroup不可以被拷贝。
2. Add和Done影响的数量必须匹配，否则会导致Wait一直阻塞。
3. 可以同时有多个协程调用Wait函数。
4. 可以借鉴sync.WaitGroup位运算方法表达多种状态。
