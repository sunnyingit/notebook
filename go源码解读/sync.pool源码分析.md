sync.pool 源码分析

## 概述

使用池化技术，可以很好的减少内存分配和GC的压力。

在做监控项目时，我们一次采集的监控数据在百万级别，如果将采集数据由字符串构建为结构体，那内存的分配压力可想而知。

好消息是，使用golang 内置包sync.pool可以很容易的创建一个对象池，使用内存池技术可以减少内存分配次数，让分配的内存重复利用。

## sync.pool使用

sync.pool的使用及其简单，只提供两个方法:
1. get: 从pool中获取一个对象
2. put: 把对象放回pool中
```
var metricPool = &sync.Pool{
    New: func() interface{} {
        return &Metric{
            record: new(Record),
        }
    },
}

// 注意池中保存的interface{}类型，需要做类型转换
func GetFromMetricPool() *Metric {
    m := metricPool.Get().(*Metric)
    return m
}


 // 放入pool之前，重置metric数据
func PutBackToMetricPool(m *Metric) {
    reSetMetric(m)
    metricPool.Put(m)
}
```


这样设计后，所有需要初始化Metric地方都使用GetFromMetricPool方法，Metric对象生命周期结束后使用PutBackToMetricPool放到pool中。

> 这里需要特别注意的是，需要保证放入池中的对象后续不在被使用, 否则有可能导致数据错乱。

sync.pool 还需要注意的点:
1. pool只适合保存临时对象，不适用连接池对象，因为GC会不定时的释放pool的对象，在go1.3版本中，会优化这块内容。
2. 每个P都持有一个大小为localSize的队列作为对象池，所以sync.pool是线程安全的，当P的可用对象使用完成后，还可以从其他P对象池中获取。
3. localSize 是可以自动扩容，缩容，不需要在初始化时传递。


## 源码分析

```
type Pool struct {
    // 禁止copy该实例
    noCopy noCopy
    local     unsafe.Pointer // 每个P都会有各自的pool, pool的个数就是P的个数
    localSize uintptr        // size of the local array
    // New optionally specifies a function to generate
    // a value when Get would otherwise return nil.
    // It may not be changed concurrently with calls to Get.
    New func() interface{}  // 初始化对象函数
}


// Local per-P Pool appendix.
type poolLocalInternal struct {
    private interface{}   // 私有临时对象，优先使用，只能被各自的P使用
    shared  []interface{} // 共享对象池，当私有对象为nil, 说则使用sharedL里面的对象
    Mutex                 // Protects shared.
}

type poolLocal struct {
    poolLocalInternal
    // Prevents false sharing on widespread platforms with
    // 128 mod (cache line size) = 0 .
    pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}


func (p *Pool) Get() interface{} {
    // 获取当前p绑定的pool
    l := p.pin()
    // 优先使用private中的对象
    x := l.private
    l.private = nil
    runtime_procUnpin()
    // private为nil，则使用shared队列的对象
    if x == nil {
        // 多个P可能同时访问，需要加锁
        l.Lock()
        last := len(l.shared) - 1
        if last >= 0 {
            x = l.shared[last]
            l.shared = l.shared[:last]
        }
        l.Unlock()
        // P的shared被用完了，则从其他P对象池中获取
        if x == nil {
            x = p.getSlow()
        }
    }
    // 如果其他P也没有可用对象，则只好初始化一个
    if x == nil && p.New != nil {
        x = p.New()
    }
    return x
}

func (p *Pool) Put(x interface{}) {
    if x == nil {
        return
    }
    // 获取到当前的p
    l := p.pin()
    // 优先把x放回到private中
    if l.private == nil {
        l.private = x
        x = nil
    }
    runtime_procUnpin()
    // 放到shared中
    if x != nil {
        l.Lock()
        l.shared = append(l.shared, x)
        l.Unlock()
    }
    if race.Enabled {
        race.Enable()
    }
}
```

## 内存对象清除
每个被使用的 sync.Pool，都会在初始化阶段被添加到全局变量 allPools []*Pool 对象中。Golang 的 runtime 将会在 每轮 GC 前，触发调用 poolCleanup 函数，清理 allPools，调用PoolClenup时候，会Stop The World。

当发生GC的时候，如果发现P正在访问其对应的l.private， 那这个l.private对象不会被清理，等下次GC再回收。



代码逻辑如下：

```
func poolCleanup() {
    // This function is called with the world stopped, at the beginning of a garbage collection.
    // It must not allocate and probably should not call any runtime functions.
    // Defensively zero out everything, 2 reasons:
    // 1. To prevent false retention of whole Pools.
    // 2. If GC happens while a goroutine works with l.shared in Put/Get,
    //    it will retain whole Pool. So next cycle memory consumption would be doubled.

    for _, p := range oldPools {
		p.victim = nil
		p.victimSize = 0
	}

	for _, p := range allPools {
		p.victim = p.local
		p.victimSize = p.localSize
		p.local = nil
		p.localSize = 0
	}

	oldPools, allPools = allPools, nil
```

在每轮 sync.Pool 的清理中，暂时不会完全清理对象池，而是将其放在 victim 中。等到下一轮清理，才完全清理掉 victim。也就是说，每轮 GC 后 sync.Pool 的对象池都会转移到 victim 中，同时将上一轮的 victim 清空掉。

为什么这么做呢？
这是因为 Golang 为了防止 GC 之后 sync.Pool 被突然清空，对程序性能造成影响。因此先利用 victim 作为过渡，如果在本轮的对象池中实在取不到数据，也可以从 victim 中取，这样程序性能会更加平滑。
