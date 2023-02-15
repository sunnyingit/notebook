# channnel源码分析

channel: 不要通过共享内存来通信，要通过通信来共享内存

目前的 Channel 收发操作均遵循了先进先出的设计，具体规则如下：

1. 先从 Channel 读取数据的 Goroutine 会先接收到数据；
2. 先向 Channel 发送数据的 Goroutine 会得到先发送数据的权利；

数据结构：
、、、
type hchan struct {
	qcount   uint
	dataqsiz uint
	buf      unsafe.Pointer
	elemsize uint16
	closed   uint32
	elemtype *_type
	sendx    uint
	recvx    uint
	recvq    waitq
	sendq    waitq

	lock mutex
}
、、、

1. qcount: Channel 中的元素个数；
2. dataqsiz: Channel 中的循环队列的长度；
3. buf: Channel 的缓冲区数据指针；
4. sendx: Channel 的发送操作处理到的位置；
5. recvx: Channel 的接收操作处理到的位置；
6. elemsize: 表示当前 Channel 能够收发的元素的大小
7. elemtype：表示当前 Channel 能够收发的元素的类型
8. sendq: 由于缓冲区空间不足而阻塞的 Send Goroutine 列表
9. recvq:由于缓冲区空间不足而阻塞的 Recived Goroutine 列表


## 管道创建

、、、
// 创建有缓存，类型为string的chan
make(chan string, 100)

// 创建无缓冲，类型为struct的chan, 省内存，尤其在事件通信的时候
make(chan struct{})
、、、


## 发送数据

发送格式如下：
、、、
ch <- data
、、、

1. 发送数据的逻辑执行之前会先为当前 Channel 加锁，防止多个线程并发修改数据。
2. 如果channel为nil则直接阻塞协程。
3. 如果 Channel 已经关闭，那么向该 Channel 发送数据时会报 “send on closed channel” 错误并中止程序。
4. 如果当前channel 的 recvq 上存在已经被阻塞的 goroutine，那么会直接将数据发送给当前 goroutine 并将其设置成下一个运行的 goroutine。
5. 如果channel还有缓冲区，则写入到缓冲区，并返回。
6. 如果不满足上面的两种情况，会创建一个 runtime.sudog 结构并将其加入 channel 的 sendq 队列中，当前 goroutine 也会陷入阻塞等待其他的协程从 Channel 接收数据。


发送数据的过程中包含几个会触发 Goroutine 调度的时机：

1. 发送数据时发现 Channel 上存在等待接收数据的 Goroutine，立刻设置处理器的 runnext 属性，但是并不会立刻触发调度；
2. 发送数据时并没有找到接收方并且缓冲区已经满了，这时会将自己加入 Channel 的 sendq 队列并调用 runtime.goparkunlock 触发 Goroutine 的调度让出处理器的使用权；

## 接收数据

從channel中接受数据方式有两种：
、、、

i <- ch

i, ok <- ch
、、、


1. 从一个空 Channel接收数据时会直接调用 runtime.gopark 让出处理器的使用权，协程阻塞并抛出异常.
2. 如果当前 Channel 已经被关闭并且缓冲区中不存在任何数据，则直接返回。
3. 如果 Channel 的 sendq 队列中存在挂起的 Goroutine，会将 recvx 索引所在的数据拷贝到接收变量所在的内存空间上并将 sendq 队列中 Goroutine 的数据拷贝到缓冲区。
4. 如果 Channel 的缓冲区中包含数据，那么直接读取 recvx 索引对应的数据；
5. 在默认情况下会挂起当前的 Goroutine，将 runtime.sudog 结构加入 recvq 队列并陷入休眠等待调度器的唤醒；


我们总结一下从 Channel 接收数据时，会触发 Goroutine 调度的两个时机：
1. 当 Channel 为空时，会阻塞当前协程，触发调度。
2. 当缓冲区中不存在数据并且也不存在数据的发送者时。


## 关闭管道

、、、
close(ch)
、、、

当 Channel 是一个【空指针】或者【已经被关闭】时，Go 语言运行时都会直接崩溃并抛出异常：

、、、
func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}
}
、、、

如何判断一个 channel 是否已经被关闭？我们可以在读取的时候使用多重返回值的方式：

、、、
x, ok := <-ch
、、、

这个用法与 map 中的按键获取 value 的过程比较类似，只需要看第二个 bool 返回值即可，如果返回值是 false 则表示 ch 已经被关闭。
