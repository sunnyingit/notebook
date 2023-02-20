# go profiling

# profiling 目标

`profiling`可以理解为性能分析，在做性能分析之前，我们要问一个问题：`性能分析目标是什么`。

简单来讲，性能分析的目标是为了找到程序的性能瓶颈。

程序性能主要体现在两个方面：cpu和memory的消耗。

通过性能分析，可以找到消耗cpu, memory最多的代码(函数)，优化这些函数后即可优化程序性能。


# profiling 采样原理

确定了性能分析目标后，我们要问第二个问题：`profiling采样原理`，这样我们能做到知其然也只其所以然。

## cpu pprofile 采样原理

go程序开启cpu性能分析后，go程序每10ms停止1次，然后记录当前正在执行的协程堆栈信息，通过堆栈信息，记录每个函数占用cpu次数，这个过程叫做采样。采样结束后，如果函数被采到的次数最多，说明这个函数累计占用cpu的时间越多，也就是需要优化的对象。

我们知道函数，函数可能调用子函数，例如：
、、、
a() {
    // do some code
    b()
    c()
    d()
}
、、、

函数`a`执行包括【本身代码执行】和【调用子函数】，执行本身代码的时被采样时间记为`flat`，函数`a`执行包括调用子函数时，被采样的时间记为`cum`(cumulative):

下面一份真实的性能分析结果:
、、、
Type: cpu  // 采用类型是cpu
Time: Jan 25, 2021 at 5:00pm (CST)
Duration: 2mins, Total samples = 2.36mins (118.00%) // 程序运行时间是2mins, 采样耗时2.36mins
    flat  flat%   sum%        cum   cum%
    13.62s  9.61%  9.61%     26.73s 18.86%  runtime.scanobject
     6.98s  4.92% 14.53%     42.76s 30.17%  runtime.gcDrain
     5.53s  3.90% 18.43%      5.62s  3.96%  runtime.cgocall
     5.08s  3.58% 22.02%      5.08s  3.58%  runtime.futex
     4.70s  3.32% 25.33%      5.91s  4.17%  runtime.findObject
     3.69s  2.60% 27.94%      3.69s  2.60%  runtime.memclrNoHeapPointers
     3.34s  2.36% 30.29%     21.36s 15.07%  runtime.mallocgc
     3.10s  2.19% 32.48%      3.13s  2.21%  runtime.pageIndexOf (inline)
     2.77s  1.95% 34.43%      3.23s  2.28%  runtime.runqgrab
     2.48s  1.75% 36.18%      2.49s  1.76%  runtime.(*lfstack).pop
、、、
每一行代表一个函数的统计结果，拿`scanobject`函数举例，从数据里面可以看到`scanobject`函数执行了13.62s，占用了百分百`9.61% = 13.62s/(2.36*60)`，这个函数还调用了子函数，最终这个函数占用cpu的时间为`26.73s`，占用比例`18.86%`。

我们通过查看`flat`和`cum`这两个指标就可以找到最耗cpu的相关函数。

`sum%`表示本行的`flat%` 加上一行的`sum%`，这个指标意义不大，可重点关注`flat`，`cum`这两个指标。

注意：cpu采样并不能分析函数的耗时时间，只能分析出函数占用cpu的比例。

## memory采样原理

默认情况下，程序每分配512KB堆内存采样一次，可以通过`runtime.MemProfileRate`配置采样频率，设置比率1将导致收集所有分配的信息，但会导致程序运行慢。

和cpu性能结果类似，函数在执行过程中可能分配内存，其调用的子函数也可能分配内存，函数本身分配的内存记为`flat`, 函数执行完成后，分配的总内存记为`cum`:

下面是一份真实的性能分析结果:
、、、
File: ae-app-300
Type: inuse_space // 采样类型是正在使用的内存
Time: Jan 25, 2021 at 5:06pm (CST)
Duration: 2mins, Total samples = 6.05MB // 采用时间是2mins，这期间采样到分配的内存是6.05MB
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 6191.91kB, 100% of 6191.91kB total
Showing top 10 nodes out of 54
      flat  flat%   sum%        cum   cum%
 1536.52kB 24.81% 24.81%  1536.52kB 24.81%  internal/profile.(*Profile).postDecode
 1064.52kB 17.19% 42.01%  1064.52kB 17.19%  icode.baidu.com/baidu/ps-aladdin/vsgo/library/dictset.SearchResourceDict
 1024.12kB 16.54% 58.55%  2048.16kB 33.08%  internal/profile.glob..func2
  518.65kB  8.38% 66.92%   518.65kB  8.38%  bytes.makeSlice
  512.05kB  8.27% 75.19%   512.05kB  8.27%  internal/profile.glob..func5
  512.02kB  8.27% 83.46%   512.02kB  8.27%  icode.baidu.com/baidu/ps-aladdin/vsgo/library/mcpack.(*decodeState).shortStringInterface
  512.02kB  8.27% 91.73%   512.02kB  8.27%  internal/profile.decodeInt64s
  512.02kB  8.27%   100%   512.02kB  8.27%  internal/profile.glob..func19
         0     0%   100%   518.65kB  8.38%  bytes.(*Buffer).Write
         0     0%   100%   518.65kB  8.38%  bytes.(*Buffer).grow
、、、

这份报告是统计使用的内存大小`inuse_space`，每一行代表一个函数的分配结果，本次性能分析一共分配内存`6191.91kB`, `postDecode`分配`1536.52kB`, 占用24.81%。


## profiling 数据采集

理解采样原理后，我们要第三个问题：`如何采样数据`。

我们的程序一般分为两种：
1. 非常驻程序。
2. 常驻程序，如web服务。

针对非常驻程序，需要在程序代码中添加采样代码：
、、、
// 获取cpu采样数据
defer profile.Start(profile.CPUProfile, profile.ProfilePath(".")).Stop()
、、、
其中，`profile.CPUProfile` 表示做cpu采样，`profile.ProfilePath(".")` 表示采样结果保存在当前目录。

当程序运行结束后，在当前目录会生成`cpu.pprof`文件。

针对常驻进程，启动服务后，需要发起请求，在请求处理时开始采样。

这种进程类型，只需要在项目中导入`import _ "net/http/pprof"`, 其他的什么都不需要做。

导入`"net/http/pprof"`包之后，`pprof`包会自动注册如下路由：
、、、
	http.HandleFunc("/debug/pprof/", Index)
	http.HandleFunc("/debug/pprof/cmdline", Cmdline)
	http.HandleFunc("/debug/pprof/profile", Profile)
	http.HandleFunc("/debug/pprof/symbol", Symbol)
	http.HandleFunc("/debug/pprof/trace", Trace)
、、、
通过访问`http://your_server_host:port/debug/pprof/profile?seconds=30` 就可以看到profile结果。

注意：构建的请求需要尽量覆盖已有的业务逻辑，这样采样的样本才会更接近用户的真实请求。

> seconds参数表示性能分析进行多长时间，默认是30s，web服务是常驻型服务，所以我们可以指定任意时间来进行性能分析，推荐不超过2min

除了获取cpu采样数据以外，还可以:
、、、
# 堆内存分配采样，可以用来分析内存分配
go tool pprof "http://your_server_host:port/debug/pprof/heap?seconds=30"

#  协程数量采样，可以用来分析协程泄露
go tool pprof "http://your_server_host:port/debug/pprof/goroutine?seconds=30"

#  阻塞采样，用来分析长时间阻塞的函数
go tool pprof "http://your_server_host:port/debug/pprof/block?seconds=30"

# 锁采样，用来分析死锁
go tool pprof "http://your_server_host:port/debug/pprof/mutex?seconds=30"
、、、

拿到采样数据后，我们下一步需要做的事情，就是解读采样数据

## 采样数据分析
拿到采样数据后，我们可以通过`go tool pprof`工具分析采样数据。

`go tool pprof`有2种方式分析采样数据。

第一种是命令行式，直接使用`go tool pprof path_to_profiling_file`。
、、、
➜  ~ go tool pprof /Users/work/pprof/pprof.ae-app-300.samples.cpu.003.pb.gz
File: ae-app-300
Type: cpu
Time: Jan 25, 2021 at 5:00pm (CST)
Duration: 2mins, Total samples = 2.36mins (118.00%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof)
、、、

执行此命令后，显示采样类型为`cpu`, 采样时间持续`2min`, 采样过程花费了`2.36min`, 因为程序会暂停，所以采样过程花费时间大于采样时间。

我们可以通过`topN`命令查看最耗时的函数，例如:
、、、
File: ae-app-300
Type: cpu
Time: Jan 25, 2021 at 5:00pm (CST)
Duration: 2mins, Total samples = 2.36mins (118.00%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof)
(pprof) top
Showing nodes accounting for 51.29s, 36.18% of 141.75s total
Dropped 1331 nodes (cum <= 0.71s)
Showing top 10 nodes out of 302
      flat  flat%   sum%        cum   cum%
    13.62s  9.61%  9.61%     26.73s 18.86%  runtime.scanobject
     6.98s  4.92% 14.53%     42.76s 30.17%  runtime.gcDrain
     5.53s  3.90% 18.43%      5.62s  3.96%  runtime.cgocall
     5.08s  3.58% 22.02%      5.08s  3.58%  runtime.futex
     4.70s  3.32% 25.33%      5.91s  4.17%  runtime.findObject
     3.69s  2.60% 27.94%      3.69s  2.60%  runtime.memclrNoHeapPointers
     3.34s  2.36% 30.29%     21.36s 15.07%  runtime.mallocgc
     3.10s  2.19% 32.48%      3.13s  2.21%  runtime.pageIndexOf (inline)
     2.77s  1.95% 34.43%      3.23s  2.28%  runtime.runqgrab
     2.48s  1.75% 36.18%      2.49s  1.76%  runtime.(*lfstack).pop
(pprof)
、、、

函数默认以`flat`排序， 如果想以`cum排序`，可执行`top -cum`：
、、、
(pprof) top -cum
Showing nodes accounting for 24.80s, 17.50% of 141.75s total
Dropped 1331 nodes (cum <= 0.71s)
Showing top 10 nodes out of 302
      flat  flat%   sum%        cum   cum%
     0.11s 0.078% 0.078%     53.50s 37.74%  runtime.systemstack
     0.37s  0.26%  0.34%     46.95s 33.12%  runtime.gcBgMarkWorker
     0.03s 0.021%  0.36%     42.87s 30.24%  runtime.gcBgMarkWorker.func2
     6.98s  4.92%  5.28%     42.76s 30.17%  runtime.gcDrain
         0     0%  5.28%     32.13s 22.67%  icode.baidu.com/baidu/ps-aladdin/vsgo/strateger/kv.(*KvStrateger).BatchRequest.func1
     0.01s 0.0071%  5.29%     32.06s 22.62%  icode.baidu.com/baidu/ps-aladdin/vsgo/module/kv.QuerySingleKvRes
    13.62s  9.61% 14.90%     26.73s 18.86%  runtime.scanobject
     3.34s  2.36% 17.26%     21.36s 15.07%  runtime.mallocgc
     0.31s  0.22% 17.47%     18.76s 13.23%  runtime.schedule
     0.03s 0.021% 17.50%     18.62s 13.14%  runtime.mcall
(pprof)
、、、

如果不想显示任何`runtime`相关的函数，可执行`top -runtime*`
、、、
(pprof) top -runtime*
Active filters:
   ignore=runtime*  // 过滤了runtime*的函数
Showing nodes accounting for 6.66s, 4.70% of 141.75s total
Dropped 581 nodes (cum <= 0.71s)
Showing top 10 nodes out of 262
      flat  flat%   sum%        cum   cum%
     2.18s  1.54%  1.54%      2.18s  1.54%  syscall.Syscall
     1.02s  0.72%  2.26%      3.70s  2.61%  icode.baidu.com/baidu/ps-aladdin/vsgo/library/mcpack.(*decodeState).object
     0.83s  0.59%  2.84%      0.83s  0.59%  syscall.Syscall6
     0.51s  0.36%  3.20%      0.51s  0.36%  icode.baidu.com/baidu/ps-aladdin/vsgo/library/mcpack.equalFoldRight
     0.40s  0.28%  3.49%      0.40s  0.28%  reflect.name.isExported (inline)
     0.36s  0.25%  3.74%      2.99s  2.11%  encoding/json.(*decodeState).object
     0.35s  0.25%  3.99%      2.04s  1.44%  encoding/json.structEncoder.encode
     0.35s  0.25%  4.23%      0.48s  0.34%  golang.org/x/text/encoding/simplifiedchinese.gbkDecoder.Transform
     0.33s  0.23%  4.47%      0.33s  0.23%  reflect.(*rtype).Kind (partial-inline)
     0.33s  0.23%  4.70%      0.34s  0.24%  sync.(*poolChain).popTail
、、、

如果想看所有函数的调用栈信息，执行`web`命令，会生成一个svg格式的文件，在浏览器中展示。
> 如果svg不能正常显示，则需要安装graphviz，在mac上执行 brew install graphviz 即可

执行`web 函数名` 只会展现所有经过函数名的调用栈。

例如，我们可以看到最耗时的函数是`runtime.scanobject`, 只需要执行 `web runtime.scanobject`就可以看到`scanobject`的调用栈信息，其他没有调用`scanobject`的函数栈会被过滤掉。


第二种方式是在浏览器中展现cpu性能分析结果，可执行`go tool pprof --http=0.0.0.0:8081    /Users/sunlili/pprof/pprof.ae-app-300.samples.cpu.003.pb.gz`。


## 火焰图和函数调用链图
