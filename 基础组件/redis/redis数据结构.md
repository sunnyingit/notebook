Redis 数据结构

# 数据结构简述
Redis 是速度非常快的非关系型（NoSQL）内存键值数据库，可以存储键和五种不同类型的值之间的映射。
键的类型只能为字符串，值支持五种数据类型：字符串、列表、集合、散列表、有序集合。
Redis 支持很多特性，例如将内存中的数据持久化到硬盘中，使用复制来扩展读性能，使用分片来扩展写性能。

| 数据类型 | 可以存储的值 | 操作 |
| :--: | :--: | :--: |
| STRING | 字符串、整数或者浮点数 | 对整个字符串或者字符串的其中一部分执行操作，对整数和浮点数执行自增或者自减操作 |
| LIST | 列表 | 从两端压入或者弹出元素，对单个或者多个元素进行修剪, 只保留一个范围内的元素 |
| SET | 无序集合 | 添加、获取、移除单个元素, 检查一个元素是否存在于集合中, 计算交集、并集、差集, 从集合里面随机获取元素 |
| HASH | 包含键值对的无序散列表 | 添加、获取、移除单个键值对, 获取所有键值对, 检查某个键是否存在|
| ZSET | 有序集合 | 添加、获取、删除元素, 根据分值范围或者成员来获取元素, 计算一个键的排名 |


## 对象系统
redis所有的数据存储都基于对象系统， 不同的数据类型对应不同的对象，包括字符串对象，列表对象，哈希对象，集合对象等。
这样Redis可以在执行命令之前， 根据对象的类型来判断一个对象是否可以执行给定的命令。
同时redis的对象系统还实现了引用计数功能，在对象不在被使用后，引用计数变成零， 从而被自动释放。 另外，通过引用计数还实现了对象共享机制，这一机制可以在适当的条件下， 通过让多个数据库键共享同一个对象来节约内存。
Redis 的对象带有访问时间记录信息，该信息可以用于计算数据库键的空转时长， 在服务器启用了 maxmemory 功能的情况下， 空转时长较大的那些键可能会优先被服务器删除。

### 对象类型和编码

Redis 使用对象来表示数据库中的键和值， 每次当我们在 Redis 的数据库中新创建一个键值对时， 我们至少会创建两个对象， 一个对象用作键值对的键（键对象）， 另一个对象用作键值对的值（值对象）。

举个例子， 以下 SET 命令在数据库中创建了一个新的键值对， 其中键值对的键是一个包含了字符串值 "msg" 的对象， 而键值对的值则是一个包含了字符串值 "hello world" 的对象：

```
redis> SET msg "hello world"
OK
```

Redis 中的每个对象都由一个 redisObject 结构表示， 该结构中和保存数据有关的三个属性分别是 type 属性、 encoding 属性和 ptr 属性：

```
typedef struct redisObject {
    // 类型
    unsigned type:4;

    // 编码
    unsigned encoding:4;

    // 指向底层实现数据结构的指针
    void *ptr;

    // 引用计数
    int refcount;
}
```


其中类型包括：
1. REDIS_STRING 字符串对象
2. REDIS_LIST	列表对象
3. REDIS_HASH	哈希对象
4. REDIS_SET	集合对象
5. REDIS_ZSET	有序集合对象

```
# 键为字符串对象，值为列表对象
redis> RPUSH numbers 1 3 5
(integer) 6

redis> TYPE numbers
list
```

对象的 ptr 指针指向对象的底层实现数据结构， 而这些数据结构由对象的 encoding 属性决定。
encoding 属性记录了对象所使用的编码， 也即是说这个对象使用了什么数据结构作为对象的底层实现， 这个属性的值可以是表 8-3 列出的常量的其中一个


|类型	  |编码	   |对象 |
| :--: | :--: | :--: |
|REDIS_STRING	|REDIS_ENCODING_INT	|使用整数值实现的字符串对象|
|REDIS_STRING	|REDIS_ENCODING_EMBSTR	|使用 embstr 编码的简单动态字符串实现的字符串对象 |
|REDIS_STRING	|REDIS_ENCODING_RAW	|使用简单动态字符串实现的字符串对象 |
|REDIS_LIST	|REDIS_ENCODING_ZIPLIST |使用压缩列表实现的列表对象 |
|REDIS_LIST	|REDIS_ENCODING_LINKEDLIST |	使用双端链表实现的列表对象 |
|REDIS_HASH	|REDIS_ENCODING_ZIPLIST |	使用压缩列表实现的哈希对象 |
|REDIS_HASH	|REDIS_ENCODING_HT |	使用字典实现的哈希对象 |
|REDIS_SET	|REDIS_ENCODING_INTSET	| 使用整数集合实现的集合对象 |
|REDIS_SET	|REDIS_ENCODING_HT |	使用字典实现的集合对象 |
|REDIS_ZSET	|REDIS_ENCODING_ZIPLIST |使用压缩列表实现的有序集合对象 |
|REDIS_ZSET	|REDIS_ENCODING_SKIPLIST |使用跳跃表和字典实现的有序集合对象 |

查看对象编码：
```
redis> SET msg "hello wrold"
OK

redis> OBJECT ENCODING msg
"embstr"
```

### 引用计数
对象的引用计数信息会随着对象的使用状态而不断变化：

在创建一个新对象时， 引用计数的值会被初始化为 1 ；
当对象被一个新程序使用时， 它的引用计数值会被增一；
当对象不再被一个程序使用时， 它的引用计数值会被减一；
当对象的引用计数值变为 0 时， 对象所占用的内存会被释放

### 对象共享

举个例子， 假设键 A 创建了一个包含整数值 100 的字符串对象作为值对象
如果这时键 B 也要创建一个同样保存了整数值 100 的字符串对象作为值对象， 那么服务器有以下两种做法：

1. 为键 B 新创建一个包含整数值 100 的字符串对象；
2. 让键 A 和键 B 共享同一个字符串对象；

以上两种方法很明显是第二种方法更节约内存。

在 Redis 中， 让多个键共享同一个值对象需要执行以下两个步骤：

1. 将数据库键的值指针指向一个现有的值对象；
2. 将被共享的值对象的引用计数增一。

目前， Redis 只对包含整数值的字符串对象进行共享。

当服务器考虑将一个共享对象设置为键的值对象时， 程序必须保证需要创建的值对象和已经存在的值对象完全一样。而共享对象的值越复杂，检测的复杂度就越高，从而消耗更多的cpu。

1. 如果共享对象是保存整数值的字符串对象， 那么验证操作的复杂度为 O(1)
2. 如果共享对象是保存字符串值的字符串对象， 那么验证操作的复杂度为 O(N)
3. 如果是其他对象，比如hash, set 那校验的算法更复杂

## 数据结构


### 字符串SDS的定义

```
type sdsdr {
    # 记录了buf中已经使用的字节数量
    int len

    # 记录buf数组中未使用的字节数量
    int free

    # 字节数组
    char buff[];
}
```

通过记录字符串长度len，可以快速读取字符串长度，通过free和buff的使用，尽可能的避免字符串修改时候内存的重新分配。


### 列表的定义
列表键的底层实现之一就是链表： 当一个列表键包含了数量比较多的元素， 又或者列表中包含的元素都是比较长的字符串时， Redis 就会使用链表作为列表键的底层实现。
```
struct listNode {
    struct ListNode *prev;
    struct ListNode *next

    # 节点的值
    void *value
} ListNode
```

多个 listNode 可以通过 prev 和 next 指针组成双端链表。

```
struct list {
     // 表头节点
    listNode *head;

    // 表尾节点
    listNode *tail;

    // 链表所包含的节点数量
    unsigned long len;

    // 节点值复制函数
    void *(*dup)(void *ptr);

    // 节点值释放函数
    void (*free)(void *ptr);

    // 节点值对比函数
    int (*match)(void *ptr, void *key);
} list
```

list 结构为链表提供了表头指针 head 、表尾指针 tail ， 以及链表长度计数器 len ， 而 dup 、 free 和 match 成员则是用于实现多态链表所需的类型特定函数：
1. dup 函数用于复制链表节点所保存的值；
2. free 函数用于释放链表节点所保存的值；
3. match 函数则用于对比链表节点所保存的值和另一个输入值是否相等。


### 字典
字典是通过hash表的方式实现
```
struct dictht {
    // 哈希表数组
    dictEntry **table;

    // 哈希表大小
    unsigned long size;

    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;

    // 该哈希表已有节点的数量
    unsigned long used;
} dictht
```

table 属性是一个数组， 数组中的每个元素都是一个指向 dict.h/dictEntry 结构的指针， 每个 dictEntry 结构保存着一个键值对。

size 属性记录了哈希表的大小， 也即是 table 数组的大小， 而 used 属性则记录了哈希表目前已有节点（键值对）的数量。

sizemask 属性的值总是等于 size - 1 ， 这个属性和哈希值一起决定一个键应该被放到 table 数组的哪个索引上面。

dictEntry的结构如下：
```
typedef struct dictEntry {

    // 键
    void *key;

    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;

} dictEntry;
```


当有两个或以上数量的键被分配到了哈希表数组的同一个索引上面时， 我们称这些键发生了冲突（collision）。

Redis 的哈希表使用链地址法（separate chaining）来解决键冲突： 每个哈希表节点都有一个 next 指针， 多个哈希表节点可以用 next 指针构成一个单向链表， 被分配到同一个索引上的多个节点可以用这个单向链表连接起来， 这就解决了键冲突的问题。

```
dictEntry1 -> dictEntry2 -> dictEntry3
```

通过key查找的时候，首先通过hash(key)找到对应的entry, 然后遍历entry列表，通过比对dictEntry的key就可以找到对应的value.


### 整数集合

整数集合（intset）是 Redis 用于保存整数值的集合抽象数据结构， 它可以保存类型为 int16_t 、 int32_t 或者 int64_t 的整数值， 并且保证集合中不会出现重复元素。


```
typedef struct intset {

    // 编码方式
    uint32_t encoding;

    // 集合包含的元素数量
    uint32_t length;

    // 保存元素的数组
    int8_t contents[];

} intset;
```

### 跳跃表

Redis 使用跳跃表作为有序集合键的底层实现之一： 如果一个有序集合包含的元素数量比较多， 又或者有序集合中元素的成员（member）是比较长的字符串时， Redis 就会使用跳跃表来作为有序集合键的底层实现。

跳跃表就是在双向列表的基础上增加多级指针，分别指向多个后续的节点，这样遍历的时候就不需要一一遍历，从而实现了跳越遍历。

例如，node1节点指向了node2,node3和node4， 如果查找node4只需要遍历一次就行。
```

node1 -> node2 -> node3 -> node4
    ------------>
    ---------------------->
```


为什么使用跳跃表
首先，因为 zset 要支持随机的插入和删除，所以它 不宜使用数组来实现，关于排序问题，我们也很容易就想到 红黑树/ 平衡树 这样的树形结构，为什么 Redis 不使用这样一些结构呢？

1. 性能考虑： 在高并发的情况下，树形结构需要执行一些类似于 rebalance 这样的可能涉及整棵树的操作，相对来说跳跃表的变化只涉及局部 (下面详细说)；
2. 实现考虑： 在复杂度与红黑树相同的情况下，跳跃表实现起来更简单，看起来也更加直观；
