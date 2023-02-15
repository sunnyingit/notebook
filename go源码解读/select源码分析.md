Select源码分析

Go 语言中的 select 也能够让 Goroutine 同时等待多个 Channel 可读或者可写，在多个文件或者 Channel状态改变之前，select 会一直阻塞当前线程或者 Goroutine。

常见用法：
、、、

select {
    // 监听ch channel
    case ret := <- ch:
        if ret < 0 {
            return ret, msg
        }
    // 加入超时处理
    case <-timer.C:
        return ret, msg
    }
}
、、、

elect 控制结构时，会遇到两个有趣的现象：
1. select 能在 Channel 上进行非阻塞的收发操作。
2. select 在遇到多个 Channel 同时响应时，会随机执行一种情况。


## 非阻塞的收发

当我们运行下面的代码时就不会阻塞当前的 Goroutine，它会直接执行 default 中的代码：
、、、
func main() {
	ch := make(chan int)
	select {
	case i := <-ch:
		println(i)

	default:
		println("default")
	}
}
、、、

常见案例：

判断tasks在执行的过程中是否出错，没有出错则直接返回nil:

、、、
errCh := make(chan error, len(tasks))
wg := sync.WaitGroup{}
wg.Add(len(tasks))
for i := range tasks {
    go func() {
        defer wg.Done()
        if err := tasks[i].Run(); err != nil {
            errCh <- err
        }
    }()
}

wg.Wait()

select {
case err := <-errCh:
    return err
default:
    return nil
}
、、、

## 总结
1. 空的 select 语句会被转换成调用 runtime.block 直接挂起当前 Goroutine；

2. 如果 select 语句中只包含一个 case，编译器会将其转换成 if ch == nil { block }; n; 表达式；首先判断操作的 Channel 是不是空的；然后执行 case 结构中的内容；

3. 如果 select 语句中只包含两个 case 并且其中一个是 default，那么会使用 runtime.selectnbrecv 和 runtime.selectnbsend 非阻塞地执行收发操作；

4. 在默认情况下会通过 runtime.selectgo 获取执行 case 的索引，并通过多个 if 语句执行对应 case 中的代码，随机选择一个case
