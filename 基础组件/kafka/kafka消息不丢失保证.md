

## 概述

越来越多的互联网公司使用消息队列来支撑自己的核心业务。
由于是核心业务，一般都会要求消息传递过程中最大限度的做到不丢失，如果中间环节出现数据丢失，就会引来用户的投诉。

那么使用 Kafka 到底会不会丢数据呢？如果丢数据了该怎么解决呢？为了避免类似情况发生，除了要做好补偿措施，我们更应该在系统设计的时候充分考虑系统中的各种异常情况，从而设计出一个稳定可靠的消息系统。

大家都知道 Kafka 的整个架构非常简洁，是分布式的架构，主要由 Producer、Broker、Consumer 三部分组成，后面剖析丢失场景会从这三部分入手来剖析。

从 Kafka 整体架构我们可以得出有三次消息传递的过程：

1. Producer 端发送消息给 Kafka Broker 端。
2. Kafka Broker 将消息进行同步并持久化数据。
3. Consumer 端从 Kafka Broker 将消息拉取并进行消费。

在以上这三步中每一步都可能会出现丢失数据的情况， 那么Kafka到底在什么情况下才能保证消息不丢失呢？

目前，kafka只对[已提交]的数据，最大限度的持久化来保证数据不丢失， 怎么理解呢?

当producer讲数据提交给kafka之后，leader broker会接受数据并持久化，然后讲数据同步给follower broker。 这个过程最大限度的保证当leader broker crash之后， follower broker依然持有数据。


##  消息丢失场景剖析

###  Producer 端丢失场景剖析

首先梳理一下Producer发送数据的逻辑：
1. Producer 端是直接与 Broker 中的 Leader Partition 交互的，以在 Producer 端初始化中就需要通过 Partitioner 分区器从 Kafka 集群中获取到相关 Topic 对应的 Leader Partition 的元数据。

2. 待获取到 Leader Partition 的元数据后直接将消息发送过去。

3. Kafka Broker 对应的 Leader Partition 收到消息会先写入 Page Cache，定时刷盘进行持久化（顺序写入磁盘）。

4.  Follower Partition 拉取 Leader Partition 的消息并保持同 Leader Partition 数据一致，待消息拉取完毕后需要给 Leader Partition 回复 ACK 确认消息。

5. 待 Kafka Leader 与 Follower Partition 同步完数据并收到所有 ISR 中的 Replica 副本的 ACK 后，Leader Partition 会给 Producer 回复 ACK 确认消息。

同时， Producer 端为了提升发送效率，减少IO操作，发送数据的时候是将多个请求合并成一个个 RecordBatch，并将其封装转换成 Request 请求「异步」将数据发送出去（也可以按时间间隔方式，达到时间间隔自动发送），所以 Producer 端消息丢失更多是因为消息根本就没有发送到 Kafka Broker 端。

导致 Producer 端消息没有发送成功有以下原因：

1. 网络原因：由于网络抖动导致数据根本就没发送到 Broker 端。
2. 数据原因：消息体太大超出 Broker 承受范围而导致 Broker 拒收消息。

所以Producer必须知道数据是否发生成功，如果失败需要重试，就像网络通信一样，为了数据的可靠性，使用了ACK来确认收到的数据，同理Kafka也是这样的设计。

Producer支持配置ACK,具体详情如下：
1. acks=0：由于发送后就自认为发送成功，这时如果发生网络抖动， Producer 端并不会校验 ACK 自然也就丢了，且无法重试。
2. acks=1: 消息发送 Leader Parition 接收成功就表示发送成功，这时只要 Leader Partition 不 Crash 掉，就可以保证 Leader Partition 不丢数据，但是如果 Leader Partition 异常 Crash 掉了， Follower Partition 还未同步完数据且没有 ACK，这时就会丢数据。
3. acks=-1 消息发送需要等待 ISR 中 Leader Partition 和 所有的 Follower Partition 都确认收到消息才算发送成功, 可靠性最高, 但也不能保证不丢数据,比如当 ISR 中只剩下 Leader Partition 了, 这样就变成 acks = 1 的情况了。


为了保证Producer不丢失数据，配置项必须是acks=-1, 同时需要满足ISR集合中的partition 副本大于1一个，因为ISR集合中默认包含了Leader Partiton。
如果此时Leader Partiton和Follower Partitoin都挂了，那也可能丢失数据，只是概率特别低。

当Ack=1或者Ack=-1的时候，有可能第一次发布给Leader Partition，Leader Partition收到数据但还没来得及ACK Producer就挂了，这样Producer会重新发送导致数据重复。

配置重试时间 retry.backoff.ms

该参数表示消息发送超时后两次重试之间的间隔时间，避免无效的频繁重试，默认值为100ms,  推荐设置为300ms。

该参数表示消息发送超时后两次重试之间的间隔时间，避免无效的频繁重试，默认值为100ms,  推荐设置为300ms。

为了解决这个问题，Broker 为 Producer 分配了一个ID，并通过每条消息的序列号进行去重，也就是说Producer发送数据时，会带上一个ID。
broker收到这个ID后，会检测最近几条日志是否保存了该ID，如果有则不会重新保存，这样就保证了数据不会重复发布。

如果Producer 重试次数达到上限依然没有发送成功，这样的话就会丢失数据，为了解决这个问题，我们需要在业务上做好补偿措施，比如把发送失败的消息记录到Redis或者mysql中，等到合适时机再次发送。


这种情况会导致消息顺序错乱，如果说一定要保证消息顺序，就只能丢弃该消息了。

对于幂等消息有个重要的问题：不能跨topic 、跨partition 保证数据一致性，如果producer 生产的消息横跨多个topic、partition, 可能会存在部分成功，部分失败的情况；另外幂等只是在单次producer 会话中， 如果pruducer 因为异常原因重启，仍然可能会导致数据重复发送，因此引入了事务解决该问题。

针对 acks = -1/ all , 这里有两种非常典型的情况：
1. 数据发送到 Leader Partition， 且所有的 ISR 成员全部同步完数据， 此时，Leader Partition 异常 Crash 掉，那么会选举新的 Leader Partition，数据不会丢失。

2. 数据发送到 Leader Partition，【部分】 ISR 成员同步完成，此时 Leader Partition 异常 Crash，会给 Producer 端发送失败标识。 剩下的 Follower Partition 都可能被选举成新的 Leader Partition，假如是未同步完成的Follower成为了Leader Partition， 那Producer会再次发送消息给新的Leader Partition，之前已经同步消息完成的Follower Partition再次同步该消息，这样就会导致消息重复。

###  Broker 端丢失场景剖析

Kafka Broker 集群接收到数据后会将数据进行持久化存储到磁盘，为了提高吞吐量和性能，采用的是「异步批量刷盘的策略」，也就是说按照一定的消息量和间隔时间进行刷盘。首先会将数据存储到 「PageCache」 中，至于什么时候将 Cache 中的数据刷盘是由「操作系统」根据自己的策略决定或者调用 fsync 命令进行强制刷盘，如果此时 Broker 宕机 Crash 掉，且选举了一个落后 Leader Partition 很多的 Follower Partition 成为新的 Leader Partition，那么落后的消息数据就会丢失。

kafka 通过「多 Partition （分区）多 Replica（副本）机制」已经可以最大限度的保证数据不丢失，如果数据已经写入 PageCache 中但是还没来得及刷写到磁盘，此时如果所在 Broker 突然宕机挂掉或者停电，极端情况还是会造成数据丢失。

我们可以通过以下参数配合来保证消息尽量不会被丢失：


4.2.1 unclean.leader.election.enable：
该参数表示有哪些 Follower 可以有资格被选举为 Leader , 如果一个 Follower 的数据落后 Leader 太多，那么一旦它被选举为新的 Leader， 数据就会丢失，因此我们要将其设置为false，防止此类情况发生。



4.2.2 replication.factor：
该参数表示分区副本的个数。建议设置 replication.factor >=3, 这样如果 Leader 副本异常 Crash 掉，Follower 副本会被选举为新的 Leader 副本继续提供服务。



4.2.3 min.insync.replicas：
该参数表示消息至少要被写入成功到 ISR 多少个副本才算"已提交"，建议设置min.insync.replicas > 1, 这样才可以提升消息持久性，保证数据不丢失。
另外我们还需要确保一下 replication.factor > min.insync.replicas, 如果相等，只要有一个副本异常 Crash 掉，整个分区就无法正常工作了，因此推荐设置成： replication.factor = min.insync.replicas +1, 最大限度保证系统可用性。

###  Consumer 端丢失场景剖析

先来分析一下Consumer的消费过程：

1. Consumer 拉取数据之前跟 Producer 发送数据一样, 需要通过订阅关系获取到集群元数据, 找到相关 Topic 对应的 Leader Partition 的元数据。
2. 然后 Consumer 通过 Pull 模式主动的去 Kafka 集群中拉取消息。
3. 在这个过程中，有个消费者组的概念（不了解的可以看上面链接文章），多个 Consumer 可以组成一个消费者组即 Consumer Group，每个消费者组都有一个Group-Id。同一个 Consumer Group 中的 Consumer 可以消费同一个 Topic 下不同分区的数据，但是不会出现多个 Consumer 去消费同一个分区的数据。
4. 拉取到消息后进行业务逻辑处理，待处理完成后，会进行 ACK 确认，即提交 Offset 消费位移进度记录。
5. 最后 Offset 会被保存到 Kafka Broker 集群中的 __consumer_offsets 这个 Topic 中，且每个 Consumer 保存自己的 Offset 进度。
6. Consumer通过Offset来指定消费消息，如果Offset出现错乱，则会导致消息丢失。

既然 Consumer 拉取后消息最终是要提交 Offset， 那么这里就可能会丢数据的, 比如

1. 拉取消息后【先提交 Offset，后处理消息】，如果此时处理消息的时候异常宕机，由于 Offset 已经提交了,  待 Consumer 重启后，会从之前已提交的 Offset 下一个位置重新开始消费， 之前未处理完成的消息不会被再次处理，对于该 Consumer 来说消息就丢失了。

2. 如果此时在提交之前发生异常宕机，由于没有提交成功 Offset， 待下次 Consumer 重启后还会从上次的 Offset 重新拉取消息，不会出现消息丢失的情况， 但是会出现重复消费的情况，这里只能业务自己保证幂等性。



我们还需要设置参数 enable.auto.commit = false, 采用手动提交位移的方式。

另外对于消费消息重复的情况，业务自己保证幂等性, 保证只成功消费一次即可，比如把消费完成的消息记录下来，这样就算Producer和Broker重复发生，Consumer消费消息之前，查询一下是否已经消费了该消息，可以避免重复消费。
