Mysql加锁分析

## 背景

MySQL/InnoDB的加锁分析，一直是一个比较困难的话题.

本文准备就MySQL/InnoDB的加锁问题，展开较为深入的分析与讨论，主要是介绍一种思路，运用此思路，拿到任何一条SQL语句，都能完整的分析出这条语句会加什么锁？会有什么样的使用风险？甚至是分析线上的一个死锁场景，了解死锁产生的原因。


### MVCC：Snapshot Read vs Current Read

MySQL InnoDB存储引擎，实现的是基于多版本的并发控制协议——MVCC (Multi-Version Concurrency Control) (注：与MVCC相对的，是基于锁的并发控制，Lock-Based Concurrency Control)。MVCC最大的好处，相信也是耳熟能详：读不加锁，读写不冲突。在读多写少的OLTP应用中，读写不冲突是非常重要的，极大的增加了系统的并发性能，这也是为什么现阶段，几乎所有的RDBMS，都支持了MVCC。
 
在MVCC并发控制中，读操作可以分成两类：快照读 (snapshot read)与当前读 (current read)。快照读，读取的是记录的可见版本 (有可能是历史版本)，不用加锁。当前读，读取的是记录的最新版本，并且，当前读返回的记录，都会加上锁，保证其他事务不会再并发修改这条记录。

1.快照读：简单的select操作，属于快照读，不加锁。(当然，也有例外，下面会分析)
select * from table where ?;

2.当前读：特殊的读操作，插入/更新/删除操作，属于当前读，需要加锁。
select * from table where ? lock in share mode;
select * from table where ? for update;
insert into table values (…);
update table set ? where ?;
delete from table where ?;

先来分析一下一个Update操作的具体流程:
当Update SQL被发给MySQL后，MySQL Server会根据where条件，读取第一条满足条件的记录，然后InnoDB引擎会将第一条记录返回，并加锁 (current read)。待MySQL Server收到这条加锁的记录之后，会再发起一个Update请求，更新这条记录之后解锁。

一条记录操作完成，再读取下一条记录，直至没有满足条件的记录为止。因此，Update操作内部，就包含了一个当前读。

### Cluster Index：聚簇索引
 
InnoDB存储引擎的数据组织方式，是聚簇索引表：完整的记录，存储在主键索引中，通过主键索引，就可以获取记录所有的列。

InnoDB聚簇索引的实现方式是B+树结构，树的叶子节点直接存储用户信息的内存地址，我们使用内存地址可以直接找到相应的行数据。

而在非聚簇索引的叶子节点上存储的并不是真正的行数据，而是主键 ID，所以当我们使用非聚簇索引进行查询时，首先会得到一个主键 ID，然后再使用主键 ID 去聚簇索引上找到真正的行数据，我们把这个过程称之为回表查询。

### 事务隔离级别

MySQL/InnoDB定义的4种隔离级别：
Read Uncommited
可以读取未提交记录。此隔离级别，不会使用，忽略。
Read Committed (RC)
快照读忽略，本文不考虑。
针对当前读，RC隔离级别保证对读取到的记录加锁 (记录锁)，存在幻读现象。
Repeatable Read (RR)
快照读忽略，本文不考虑。
针对当前读，RR隔离级别保证对读取到的记录加锁 (记录锁)，同时保证对读取的范围加锁，新的满足查询条件的记录不能够插入 (间隙锁)，不存在幻读现象。
Serializable
从MVCC并发控制退化为基于锁的并发控制。不区别快照读与当前读，所有的读操作均为当前读，读加读锁 (S锁)，写加写锁 (X锁)。
Serializable隔离级别下，读写冲突，因此并发度急剧下降，在MySQL/InnoDB下不建议使用。



### sql加锁分析

SQL1：select * from t1 where id = 10;
SQL2：delete from t1 where id = 10;

组合一：id列是主键，RC隔离级别。

id是主键，Read Committed隔离级别，给定SQL：select快照读不加锁，delete from t1 where id = 10; 只需要将主键上，id = 10的记录加上X锁即可。

组合二：id唯一索引+RC

这个组合，id不是主键，而是一个Unique的二级索引键值。那么在RC隔离级别下，delete from t1 where id = 10; 需要加什么锁呢？

假设组建索引为name列，此组合中，id是unique索引，而主键是name列。

此时，加锁的情况由于组合一有所不同。由于id是unique索引，因此delete语句会选择走id列的索引进行where条件的过滤，在找到id=10的记录后，首先会将unique索引上的id=10索引记录加上X锁，同时，会根据读取到的name列，回主键索引(聚簇索引)，然后将聚簇索引上的name = ‘d’ 对应的主键索引项加X锁。

为什么聚簇索引上的记录也要加锁？试想一下，如果并发的一个SQL，是通过主键索引来更新：update t1 set id = 100 where name = ‘d’; 此时，如果delete语句没有将主键索引上的记录加锁，那么并发的update就会感知不到delete语句的存在，违背了同一记录上的更新/删除需要串行执行的约束。

组合三：id不是索引+RC
相对于前面三个组合，这是一个比较特殊的情况。id列上没有索引，where id = 10;这个过滤条件，没法通过索引进行过滤，那么只能走全表扫描做过滤。对应于这个组合，SQL会加什么锁？或者是换句话说，全表扫描时，会加什么锁？这个答案也有很多：有人说会在表上加X锁；有人说会将聚簇索引上，选择出来的id = 10;的记录加上X锁。

由于id列上没有索引，因此只能走聚簇索引，进行全部扫描。从图中可以看到，满足删除条件的记录有两条，但是，聚簇索引上所有的记录，都被加上了X锁。无论记录是否满足条件，全部被加上X锁。既不是加表锁，也不是在满足条件的记录上加行锁。


## 死锁原理与分析

Transation1:
```
SQL1：select * from t1 where id=1 for update;
SQL2：delete from t1 where id=5;
```

Transation2:
```
SQL1：delete from t1 where id=5;
SQL2：select * from t1 where id=1 for update;
```

mysql进行加锁是需要满足2PL的，在事务中先对于满足的记录进行加锁，事务完成后再进行解锁；

对T1来讲，先对于id=1的行加锁，对T2来讲，先对id=5的行加锁；
T1执行delete操作是需要等待T2释放锁， T2也在等待T1释放锁，于是出现了死锁；

死锁的发生与否，并不在于事务中有多少条SQL语句，死锁的关键在于：两个(或以上)的Session加锁的顺序不一致。

例如T1需要对记录[h1, h2]加锁，另一个T2需要对记录[h2, h1]进行加锁，这样也能出现死锁。

而使用本文上面提到的，分析MySQL每条SQL语句的加锁规则，分析出每条语句的加锁顺序，然后检查多个并发SQL间是否存在以相反的顺序加锁的情况，就可以分析出各种潜在的死锁情况，也可以分析出线上死锁发生的原因。
