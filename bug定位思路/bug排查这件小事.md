关于bug排查的那些事

每当出现线上case时，你是否会心惊肉跳，无从下手，是否会羡慕大佬气定神闲，一顿操作完美定位bug，收割全场的目光。
解决问题时，最重要原则是：大胆怀疑，小心求证，利用各种工具验证自己的想法，直到最终找到根因。


## bug归类

线上问题大致可以分为：
1. 业务问题，比如配置文件不存在，访问数据不存在，业务逻辑不符合预期等。
2. 代码问题，比如代码语法问题，例如空指针，并发问题，死锁，内存泄露，协程泄露，进程core，进程panic等。
3. 资源问题，比如cpu，内存，磁盘io等资源不足。
4. 网络问题，比如远程服务连接失败，请求耗时严重，请求包丢失，重试配置不当。
5. 进程问题，比如进程启动失败，假活进程，进程保活脚本逻辑有问题。
6. 未知问题，难以定位，难以复现。

这几类问题并非各自独立，而是相互影响，比如代码有内存泄露会导致内存不足，出现OOM的情况; 代码有性能问题，导致cpu利用率异常，cpu利用率异常也会导致网络连接出现问题。这就像多米诺效应，一个问题会导致另外一个问题，最重要的是要定位到问题的根因。


## 日志
线上问题的定位必须依赖日志系统，日志系统记录了一个请求生命周期的关键信息。

一个完整的日志系统包括：
1. 业务日志，包括正常日志，记录所有请求; 错误日志，记录失败的请求。 (个人不推荐加warning日志)。
2. 进程日志，包括进程启动日志，进程panic时的栈信息，进程core信息。
3. rpc请求日志，包括请求远程服务的信息。

业务日志格式一般包括以下必要的字段：
1. 请求时间，精确到秒；
2. logid，标示请求的唯一id，用于跟踪请求生命周期内访问的所有服务;
3. 状态码，标记请求是否正常；
4. 请求耗时，单位一般是毫秒；
5. 本机ip，记录所在机器；
6. 错误信息，比如配置不存在，访问的数据不存在等；
7. request-length，记录请求包体的大小
8. response-length，记录返回包体的大小
9. http-method，记录http请求的方法


进程日志信息一般包括：
1. 服务启动时间，启动时写入的必要信息
2. 服务panic时，记录的堆栈信息

rpc请求日志处理包括业务日志所有字段以外，还必须包括：
1. 请求的远程服务名，方法，可能还包括请求参数；
2. 请求耗时，包括connection耗时，read耗时，write耗时
3. 重试次数；
4. 远程服务ip；
5. message信息，记录了请求正常或者错误时抛出的信息；

我们将结合日志信息，分享问题排查思路。假如我们有一个错误日志app-name.log.2022012302,内容如下：

```
ERROR: 2022-01-23 02:05:00 qid[2823482873] qid64x[a84aedf9] logid[2823482873] resp_code[0] cost[0] idc[idc-name] module[app-name] uri[/ab/cd?d=f] method[get] status[200] host[10.229.174.144:8210] client_ip[10.128.97.13] remote_ip[10.128.97.13] optime[1642874700393] refer[] protocol[HTTP/1.1] message [conf-file is not exsited]
```

### 业务bug
业务问题相对比较好排查。遇到业务问题时，例如配置文件不存在，只需要通过调用方提供的logid和调用时间，此时只需要根据提供的logid找到对应的业务错误日志，查看日志的状态码和错误信息就可以定位到bug。

常用命令：grep logid app-name.log.2022012302，查看对应的message基本就可以定位到相关问题。

## 代码问题
遇到代码问题，例如语法不当导致请求失败，种情况也只需要使用 grep logid 进程日志即可。一般语法错误也会打到日志中，我们根据日志就可以定位到问题。

第二种情况，代码问题可能会导致进程crash down，由于进程可以被supervisor等进程托管，会自动拉起， 所以，我们需要确认进程是否重启：

```
> ps -aux | grep app-name
USER     PID    %CPU  %MEM   VSZ    RSS TTY   STAT START  TIME COMMAND
work     125981  2.7  0.0 14322944 175356 ?   Sl   Jan22  72:31 app-name
```


以上输出中，TIME：进程运行的时间；START：进程启动时间
如果这两个指标不符合预期，则表示进程有重启，有重启则表示进程曾经crash down。
我们再检查进程panic日志，如果进程有crash，则进程日志应该会保存当时栈信息。

栈的信息一般给告知具体的语法错误，通过`dlv attach pid`命令，可以单步跟踪进程，根据栈信息定位到出错函数，然后排查出问题。

如果怀疑代码有内存泄露问题，可以通过 `pidstat -r` 确认进程是否增长，可以通过`grep -i 'killed process' /var/log/messages` 确认进程是否有OOM。

如果有OOM，对于Go进程而言，一般有协程阻塞，变量循环引用导致无法释放，申请超大内存，网络连接没有释放，文件句柄没有释放等情况。

如果有协助阻塞，进程对应的线程也会持续增加，通过`cat /proc/pid/status`查看进程下面的线程数：
```
cat /proc/223/status
....
VmData:    71556 kB
VmStk:       544 kB
VmExe:     10304 kB
VmLib:    120524 kB
VmPTE:       448 kB
VmSwap:        0 kB
Threads:        1 // 线程的数量
SigQ:   2/94992
```
如果线程数过大，则可以怀疑是协程阻塞，一般协程阻塞主要是协程的写入或者读取没有设置超时时间。

如果确定了内存泄露，可以使用`go pprof`采集两次分配的内存，然后通过`go tool pprof -diff_base  xxx.alloc_space.inuse_objects.inuse_space.001.pb.gz xxx.alloc_objects.alloc_space.inuse_objects.inuse_space.002.pb.gz` 对比后，查找到分配内存最多的函数大概率是内存泄露的函数。


遇到代码并发读写问题，可以通过`go run  --race`检测出来：

```
package main
import "fmt"

func main() {
    i := 0
    go func() {
        i++ // write
    }()
    fmt.Println(i) // concurrent read
}
====================================

$ go run -race main.go
0
==================
WARNING: DATA RACE
Write by goroutine 6:
  main.main.func1()
      /tmp/main.go:7 +0x44

Previous read by main goroutine:
  main.main()
      /tmp/main.go:9 +0x7e

Goroutine 6 (running) created at:
  main.main()
      /tmp/main.go:8 +0x70
==================
Found 1 data race(s)
exit status 66
```

### 系统资源不足

排查系统故障问题之前，我们需要确定故障发生的时间，最好能精确到时分秒，这是非常重要的一步。怎么定位故障时间呢，可以通过监控系统 (一般大型系统都有相关监控指标)。

我们可以通过如下指标定位服务故障发生时间：
1. 机房总tatol_qps和单机抽样qps 是否突然上涨。
2. 服务的存活实例个数是否突然下降。
3. 机房平均响应时间和抽样单机平均响应时间是否突然上涨。
4. 机房平均内存使用大小是否上涨。
5. 机房平均线程数和抽样实例线程数是否上涨。

通过这些指标确定好故障时间后，我们能得到一个初步的结论： 是否因为qps上涨导致服务不可用。
假如发现机房的总qps突然上涨，抽样实例的qps也上涨，我们在找一台故障机器，查看该机器的系统资源是否支持暴涨的qps。

问题来了，我们怎么判断该机器是因为资源不够导致实例crash down呢，回答这个问题前，我们需要知道，机器资源大致可以分几类：cpu, 内存，磁盘IO，网络。

首先是看cpu的指标：

1. %id: cpu的空闲时间比率。 这个指标的意思是，在cpu利用率采样期间内，cpu既没有参与运算(参与运算会增加system, user比率的值)， 也不处于IO阻塞状态(增加iowait比率的值)，如果cpu_idle的值小于20%，则基本可以判断出cpu资源不够。

2. %wa: cpu处于io wait的时间比率。要想理解这个指标，需要清楚一点，cpu在提交IO请求后(read&write)，线程会挂起,  当IO完成后，再通过中断通知线程转到就绪队列，等待被再次调度。在CPU等待IO完成期间，如果没有执行其他线程任务，那这段时间就是cpu的iowait时间。
所以，io_wait是一种特殊的idle。当iowait的值大于50%，可能会有磁盘IO问题。

3. %us: 用户态cpu利用率，这个指标的意思是，在cpu采样期间内，cpu执行用户代码花费的时间。
4. %sy:  内核态cpu利用率，这个指标的意思是，在cpu采样期间内，cpu执行系统代码花费的时间。

当qps上升时，%id指标会下降，%us, %sy, %hi指标会同步上升，如果这些变化的时间点和故障发生的时间点同步是，则可以推断出cpu资源不够。

内存指标：
1. mem_used: 内存使用量
2. mem_used_rate: 内存使用率 (使用量 / 内存总量)

当qps飙升时，这两个指标都会上升，mem_used_rate > 80% 可能会导致oom，从而导致实例crash down。 如果这些变化的时间点和故障发生的时间点同步，可以推断出内存资源不够。

### OOM

假如机房的总qps并没有上升，抽样实例的qps有少量上涨，服务的存活实例个数突然下降。

遇到这种情况，首先我们需要做的事情是，判断实例为什么会crash down。 首先，我们找一台可疑的机器登陆上去，找到服务对应的实例进程。

在判断实例会crash down之前，我们确认实例曾经crash down过. 这里有个细节，一般我们服务会通过supervisor等守护工具启动，当实例crash down之后，会自动被supervisor拉起来，但是我们可以通过实例启动时间和已经运行的时间来判断实例是否重启。

指令为：

```

git:(master) ps aux | grep ae-app-179

work 13897 0.9 0.7 1831288 189928 pts/3 Sl+ 20:00 0:33 ./bin/ae-app-179

```

20:00 表示实例启动时间，0:33 表示已经运行的时间。如果这两个时间和预期不匹配，则说明实例有被重启。

如果确定实例有被重启，则说明我们找的机器的实例出现过问题，则可进一步排查。

首先怀疑是不是OOM(out of memory)导致。怎么验证呢，有两种方法：

```

// 查询/var/log/message系统错误日志，当程序oom时，会被记录到此文件中

grep -i 'killed process' /var/log/messages

// 或者是dmesg
dmesg | egrep -i 'killed process'
```

通过这两个命令可以定位到是否是OOM导致, 假如是OOM导致，关于OOM的诱因，我们可以做如下猜想：

1. 频繁申请对象，未及时释放，导致内存泄露。
2. channel，锁，rpc超时等机制导致协程阻塞，已用协程无法释放导致内存泄露，这种情况会伴随线程数同时飙升。
3. rpc调用超时设置不合理，导致一个请求的处理时间过长，后续待请求累计过多，消耗过多内存。
4. rpc调用重试设置不合理，也导致一个请求的处理时间过长，后续待请求累计过多，消耗过多内存。
5. io性能瓶颈导致请求时间过长，后续待请求累计过多，消耗过多内存。
6. 触发了特殊逻辑，申请了过多内存，导致OOM。


猜想1：如果只有一个机房出现服务不可用，其他机房并没有问题。同时看出实例正常运行期间时，如果内存没有直线上升，则可排除申请对象非释放导致问题。

猜想2： 如果qps没有明显增加，但线程数明显增加，则验证猜想2。 怎么判断qps没有明显增加，而线程数增加呢?

第一步通过日志，grep故障时间点日志的记录条数可以得到qps，例如查看故障时间11:24:27点qps

```
grep "11:24:27" log_file | wc -l
```

在对比昨天当前时间的qps的，就可以判断qps是否明显增加。

第二步，查看实例的线程数，使用命令：
```
pidstat -T
```

如果线程对比其他实例有明显增加，则可以判断阻塞导致了线程数量增加。

猜想3，4：rpc超时不合理或者重试过多，也会导致协程释放不及时。这个问题只需要查看rpc日志：
1. 通过 rpc错误码看下错误是否增加。
2. 查看rpc的消耗时间是否过长，一般大于300ms的rpc请求属于超长时间了。

使用命令如下：

```

// rpc请求日志如下

 NOTICE: 2021-04-13 20:00:26 RAL app_name[app-name] service[service-name] req_len[220] res_len[2950] status_code[0] retry[0/1] cost[63.818] api[QueryService] logid[3529816953] protocol[PRPC] balance[RoundRobin] user_ip[] local_ip[10.12.196.131] idc[test] remote_ip[10.41.105.85:8025] remote_idc[hbe] remote_host[] uniqid[35298169530] talk[35.211] connect[28.594] write[0.127] read[35.076] pack[0.218] unpack[0.069] connect[32]

// 查看故障时间内status_code !=0 的记录数有多少

cat rpc.log | grep "11:24:2*" | grep -oP  ’status_code\[[0-9]*\]‘ | sort | uniq -c | sort -rn | grep -v 'status_code\[0\]'

cat ae-app-548.log.2021123111  | awk '{ret=match($0, /result_code\[([0-9]+)\].*vsgo_cost\[([0-9]+)\].*cost\[([0-9]+)\]/, any); if (any[1]== 0) print any[1], any[2], any[3]}' | awk '{print $2}' | awk '{a+=$1}END{print a/NR}'



// 查看故障时间内，rpc服务对应的超时时间，重试次数, 超时时间大于50ms。
cat rpc.log | awk '{ret=match($0, /service(\[[a-zA-Z\-]*\]).*(retry\[[0-9/]*\]).*cost\[([0-9/\.]*)\]/, any); if (any[2] > 50) print any[1], any[2], any[3]}'

```
通过日志发现了rpc的相关问题，则需要重新配置合理的参数，其次需要验证rpc远程服务是否正常。

如何根据这些指标判断io瓶颈呢？

首先需要明确的是，并没有一个明确的指标衡量io瓶颈，我们需要结合多个指标才能分析。

一般%await > 50，%util 接近100%，%avgqu-sz > 30, 同时cpu的iowait的有明显的飚高，则可以证明io性能有问题。

这里需要搞清楚一个误区，%util接近100%不一定表示io有性能问题，因为%util 有io处理的时的比例，处理1个io请求和处理10个io请求%util的值可能一样，因为磁盘io可以并行处理io请求。


猜想6： 特殊逻辑可能导致突然申请特别大的内存，比如程序触发coredump，或者sql查询语句返回特别多条数记录，这种问题可以通过流量复现定位到，流量复现工具推荐是tcpcopy。


### awk和elk

如果日志系统采用了ELK，那统计日志的相关指标会非常简单，如果没有使用ELK系统，那我们只能通过相关命令，常用格式：
```
// 统计状态码不等于0，各个状态的个数
cat ae-app-300.log.2021110113|awk '{ret=match($0,/result_code\[([0-9]*)\]/,ary); if(0<ret){value[ary[1]]+=1; total+=1;}}{for(key in value){print value[key],key} print total,"total";}'|sort -rn|head -n 10

// 统计各个状态码下耗时情况
cat ae-app-337.log.2021112613|awk '{ret=match($0,/cost\[([0-9]*)\]/,ary); if(0<ret){idx+=1; value[idx]=int(ary[1]);}}END{asort(value,value2);idx2=int(idx*0.97); print idx, idx2, value2[idx2];}
```
