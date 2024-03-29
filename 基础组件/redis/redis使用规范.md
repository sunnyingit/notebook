Redis 使用说明

## 缓存的更新套路
读操作：先读缓存，如果缓存没有数据，则从数据库读取
写操作：先写数据库，写数据库成功后，在使缓存失效
错误套路: 千万不要先删除缓存,在更新数据库，否则删除之后有个读操作发现缓存么有数据，则从数据库读取数据(旧数据)更新到缓存。

## redis 单进程运行模式
redis-server采用异步非阻塞模式，因为是单线程的，所以只会有一个命令被执行，所以大部分的处理都是原子性的。
因为redis命令是原子性，可以用来做并发控制(多个客户端同时执行decr),每次只会有一个客户端成功。

```
# 原子操作, 可以用来做并发控制
redis.decr(1)

# 如果是两个命令的话，是不能保证原子操作的
v = redis.get(k)
redis.set(v-1)
```

因为redis是单线程的，所以如果某个命令被阻塞，则redis整个服务都会被阻塞

高危命令： keys(), hgetall() ，可能导致服务阻塞，使用SCAN,HSCAN,ZSCAN代替

## Redis使用规范
1. key必须有相应的前缀： 'key_prefixe:{}.format(key)'
2. key必须设置缓存时间
3. key建议加版本信息，在不同的版本中储存的value的结构可能不一样

## 热键和雪崩
热键：在某些场景中，某些key会被经常访问到，导致某台redis的负载过大

如何判断热键呢，主要是通过key的访问次数和超时时间来判断，如果超过某个阈值则可以判定为热键。
解决：使用取模的方式可以把热键散列到多台服务器中

hash_key不能配置，必须不同的实例hask_key不一样，所以正好可以获取redis client所在的机器ip。
如果这个机器被摘，那么对于的key都无法再被访问，所以最好加入超时时间。
```
# 根据本机的ip取模，这样请求到本机的key都会散列某台服务器上
# 在分布式环境中，web机器有很多，所有key会被散列到不同的redis机器上
ip_address = socket.gethostbyname(socket.gethostname())
hash_key = ip2long(ip_address) % 16
cache_key = 'key:{}:{}'.format(hash_key, key)
```

雪崩： redis的key在统一时间失效，如果在一次请求中，需要访问三个key，但是这三个key的缓存失效时间都一样，导致所有的请求都打到数据库
解决： 设置在key的缓存时间加一个随机数，避免所有的key都在同一时间失效

在golang中，使用NewSingleFlight组件可以避免多次请求下游，设计思路类似。


## 内存淘汰机制

Redis 采用的是定期删除+惰性删除策略。
因为Redis是单进程的，所以在redis定期删除缓存期间，不提供服务。

定时删除：
Redis 默认每个 100ms 检查，有过期 Key 则删除。需要说明的是，Redis 不是每个 100ms 将所有的 Key 检查一次，而是随机抽取进行检查。

如果只采用定期删除策略，会导致很多 Key 到时间没有删除

惰性删除策略: 当访问某个key，首先检查这个key是否过期，过期则删除。

比如redis机器只有50M内存，如果redis的key大于50M，则需要清除相应的key，保证新的key有空间使用

清除策略一般采用 allkeys-lru：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 Key

## 事务和Watch命令

事务可以保证redis操作的一致性和原子性。

事务状态则是以一个事务为单位，执行事务队列中的所有命令：除非当前事务执行完毕，否则服务器不会中断事务，也不会执行其他客户端的其他命令

首先需求了解的是Redis事务只支持单机，如果在事务中操作的key发布在不同的机器上，则不可以使用事务(所以redis的事务很鸡肋，因为一般线上都采用redis集群的方式)

常用命令:
MULTI：开启事务, 开启事务后，client发送的命令都被server存入到队列中
DISCARD：取消事务，清空队列中的命令
EXEC：执行事务命令，执行server队列的命令
Watch:在事务开启前client watch的相应的key， 如果在事务EXEC之前，此key被其他客户端修改，则事务失败

## Redis订阅和发布

SUBSCRIBE 命令可以让客户端订阅任意数量的频道，如果有三个客户端订阅了 channel1
当有新消息通过 PUBLISH 命令发送给频道 channel1 时， 这个消息就会被发送给订阅它的三个客户端
当有新消息发送到频道时，redis-server遍历频道（键）所对应的（值）所有客户端，然后将消息发送到所有订阅频道的客户端上
所以客户端需要注册一个回调函数，如果有消息过来，触发回调函数。

## 持久化
我们知道， redis的数据都以数据结构的形式保存在内存中，为了让redis重启后依然可用，就必须把内存中的数据写入到硬盘，redis提供了RDB和AOF方式。

### RDB

在redis运行期间，RDB程序把当前redis内存快照数据保存到磁盘中，在 Redis 重启动时， RDB 程序可以通过载入 RDB 文件来还原数据库的状态。

redis提供两个命令SAVE和BGSAVE

SAVE：主进程直接阻塞，直到RDB文件保存位置，在主进程阻塞期间，服务器不能处理客户端的任何请求

BGSAVE：Fork出一个子进程，子进程和父进程共享内存信息，所以子进程可以保存redis内存信息，redis父进程可以继续提供服务

BGSAVE和BGWRITEAOF程序不能同时执行。

如果发现发现也有旧RDB文件，则替换为新的RDB文件

RDB的格式：

+-------+-------------+-----------+-----------------+-----+-----------+
| REDIS | RDB-VERSION | SELECT-DB | KEY-VALUE-PAIRS | EOF | CHECK-SUM |
+-------+-------------+-----------+-----------------+-----+-----------+

RDB-VERSION: RDB的版本信息
SELECT-DB: 数据据库信息
KEY-VALUE-PAIRS: 数据库中所有的key-value对
当 Redis 服务器启动时， rdbLoad 函数就会被执行， 它读取RDB文件， 并将文件中的数据库数据载入到内存中。


save执行策略：save <seconds> <changes>  在指定秒数和数据变化次数之后把数据库写到磁盘上
1. save 300 10 # 300秒（5分钟）之后，且至少10次变更
2. save 60 10000  # 60秒之后，且至少10000次变更

### AOF
Redis将数据库所有执行的命令都记录到AOF文件里以此达到记录数据库状态的目的，有点像数据库的binlog。

AOF的流程分为三步:
1. Redis将执行完的命令，命令的参数等信息都发送到AOF程序中
2. AOF程序根据接收到的命令数据，将命令转换为网络通讯协议的格式，然后将协议内容追加到服务器的AOF缓存中
3. 文件写入和保存，AOF 缓存中的内容被写入到 AOF 文件末尾，如果设定的 AOF 保存条件被满足的话， fsync 函数或者 fdatasync 函数会被调用，将写入的内容真正地保存到磁盘中。


文件写入和保存：
WRITE：根据条件，将 aof_buf 中的缓存写入到AOF文件，这个文件记录的是redis的命令，不是key对应的value值。
SAVE：根据条件，调用 fsync 或 fdatasync 函数，将AOF文件保存到磁盘中。

Redis 目前支持三种 AOF 保存模式，它们分别是：

1. AOF_FSYNC_NO ：不保存
2. AOF_FSYNC_EVERYSEC ：每一秒钟保存一次，write操作由主进程处理，Save操作由子线程处理
3. AOF_FSYNC_ALWAYS ：每执行一个命令保存一次，write和save都是由主进程处理

AOF 重写的实现

AOF文件记录的是redis操作命令，比如某个key被操作了100次，那么AOF会有100记录，AOF 重写就是计算出key的最终值

AOF重写的操作是有子进程执行的，子进程在进行 AOF 重写期间， 主进程还需要继续处理命令， 而新的命令可能对现有的数据进行修改， 这会让当前数据库的数据和重写后的 AOF 文件中的数据不一致，为了解决这个问题，AOF加了一个缓冲。

换言之， 当子进程在执行 AOF 重写时， 主进程需要执行以下三个工作:
1. 处理命令请求。
2. 将写命令追加到现有的 AOF 文件中。
3. 将写命令追加到 AOF 重写缓存中。

缓冲区的做法虽然提高了效率， 但也为写入数据带来了安全问题， 因为如果计算机发生停机， 那么保存在内存缓冲区里面的写入数据将会丢失。
为此， 系统提供了 fsync 和 fdatasync 两个同步函数， 它们可以强制让操作系统立即将缓冲区中的数据写入到硬盘里面， 从而确保写入数据的安全性。


## 复制功能 Master和Salve的同步

同步是将从服务器的状态同步到和主服务器的状态，使用RDB的方式同步，过程如下:

1. 从服务器向主服务器发送 SYNC 命令。

2. 收到 SYNC 命令的主服务器执行 BGSAVE 命令，在后台生成一个 RDB 文件， 并使用一个缓冲区记录从现在开始执行的所有写命令。

3. 当主服务器的 BGSAVE 命令执行完毕时， 主服务器会将 BGSAVE 命令生成的 RDB 文件发送给从服务器，

4. 从服务器接收并载入这个RDB文件，将自己的数据库状态更新至主服务器执行 BGSAVE命令时的数据库状态

5. 主服务器将记录在缓冲区里面的所有写命令发送给从服务器， 从服务器执行这些写命令，将自己的数据库状态更新至主服务器数据库当前所处的状态。

命令传播：主从同步后，如果主服务器处理的新的命令，则需要把命令同步到从服务器，从服务器执行命令后，状态和主服务器保持一致

## Pipeline
可以将多次IO往返的时间缩减为一次，注意 pipeline并不能保证原子性

```
with client.pipeline(transaction=False) as pipe:
    for meta in metas:
        key = self.format_key(meta.id)
        pipe.set(key, meta.integer, ex=int(meta.ttl))
    pipe.execute()
```


## Redis和分布式锁

在分布式系统里面，可能有多个client都需要竞争同一种资源，所以需要锁来保证一次只能有一个client能获取到锁。

分布式竞争资源的场景：
一个订单，客户正在前台修改地址，管理员在后台同时修改备注。地址和备注字段的修改，都必须正确更新，这二个请求同时到达的话，如果不借助db的事务，很容易造成行锁竞争，但用事务的话，db的性能显然比不上redis轻量。

解决思路：A,B二个请求，谁先抢到分布式锁（假设A先抢到锁），谁先处理，抢不到的那个（即：B），在一旁不停等待重试，重试期间一旦发现获取锁成功，即表示A已经处理完，把锁释放了， 这时B就可以继续处理了。

一个好的分布式锁必须保证：
1. 安全性，任意情况下只能有一个客户端能拿到锁
2. 避免死锁，如果redis-server机器挂了，或者网络断了，客户端拿到的锁一定能释放，保证其他客户端一定能拿到锁
3. 容错性，只要锁服务中大部分节点存活，就可以拿到锁。

redis怎么保证这三个特性：

```
SET resource_name random_value NX PX 30000
```

1. random_value必须是随机的，这个随机数可以保证锁的安全性
2. NX：表示只能当resource_name不存在的情况下才能设置成功
3. PX：表示锁的有效时间，时间到了会自动释放锁

### 获取锁
当多个客户端同时执行set操作的时候，因为redis是单进程的，同时只会有一个客户端执行成功，当某个客户端执行成功后，其他的客户端就会失败,NX表示只有key不存在的情况下才能set成功

这样，set成功的client就获取到锁了

### 释放锁
如果某个client执DEL resource_name，就可以解锁，这肯定是不行的，所以删除的key之前必须要比对Value，只有key相同的情况下才可以删除

还有一种情况，如果ClientA拿到锁，但是ClientA因为某些操作被阻塞了，锁超过了有效自动释放了，于是ClientB可以拿到锁。

大概ClientA又可以执行并执行完成后，会释放锁，于是会把ClientB的锁释放掉

所以，释放锁之前，必须要compare一下，看看value对不对的上

这就是CAS(compare and set)的方式

### 特殊情况

如果clientA获取到锁，此时master redis-server挂了, 在挂之前没有把命令同步salve redis-server， 那么另外一个客户端b就可以在salve redis-server拿到锁了。

这样同时就有两个客户端拿到锁，类似的问题是为了保证高可用性，会使用多个redis-server服务获取锁，如果某个server挂了，会影响锁的获取
使用REDlock锁算法可以解决此问题。REDlock原理就是当拿到一般服务器数量的锁，表示锁获取成功。

## Redis集群

Redis集群有两种方案，客户端方案和服务器端方案。

客户端方案：有客户端配置服务器列表，根据算法获取其中一条服务器访问

缺点:
1. 无法正常使用 MSET、MGET 等多种同时操作多个 Key 的命令，需要使用 Hash tag 来保证多个 Key 在同一个分片上
2. 迁移key需要停机处理
3. 由于每个客户端都要与所有的分片建立池化连接，客户端基数过大时会造成 Redis 端连接数过多

服务器端方案： redis cluster(官方) & Twemproxy(推特)

服务器端方案大都使用一致性哈希（Consistent Hashing）算法来散列redis服务器。

## 慢日志
慢日志，记录查询较慢的命令
slowlog-log-slower-than：选项指定执行时间超过多少微秒（1 秒等于 1,000,000 微秒）的命令请求会被记录到日志上
slowlog-max-len：选项指定服务器最多保存多少条慢查询日志，超过此条数会删除旧的慢日志


## 案例分析

跟风弹幕：每个房间中，在Xs内，同⼀条弹幕被Y个⽤户发送；

使用有序集合保留每个房间内的所有弹幕，格式为：

zadd 房间 弹幕发送时间 弹幕内容+uid

每次发送一次弹幕，都通过ZRANGE命令把房间内所有的弹幕都读出来，然后判断是否有跟风弹幕生成，同时清除超过Xs内的弹幕。
这样保证了有序集合里面元素的有限。

当跟风弹幕生成之后，使用setNx保存跟风弹幕并广播出去，跟风弹幕生成之后不需要判断是否有新的跟风弹幕知道跟风弹幕超时。
