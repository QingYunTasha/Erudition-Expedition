# Redis
## basic data structure
### string
ArrayList
分配空間方式
if < 1M:
    double
else:
    add 1M

maximum 512M

### list
if data small:
    ziplist
else:
    quicklist == ziplist + linkedlist

### hash
HashMap == hash + linkedlist

use progressive hash: non block many times

### set
hashSet

### zset
sortedSet + hashMap
唯一性 ＋ 排序
JumpList

### time out issue on lock
do not use locker on longer task
use Lua to atomic operations to compare locker ID and lock & unlock

### reentrant lock
可重入性是指线程在持有锁的情况下再次请求加锁，如果一个锁支持同一个线程的多次加
锁，那么这个锁就是可重入的

### message queue
no ack ensurance
Consider idle when block for long time can cause disconnect

### 鎖衝突處理
1、直接抛出异常，通知用户稍后重试；
2、sleep 一会再重试；
可能會因為deadlock造成thread block
3、将请求转移至延时队列，过一会再试；
可以用zset實現，並用lua scripting優化zrangebyscore & zrem為atomic operation

### bitmap
可以用bitfield直接操作最多64bit

### hyperLogLog
去重複 記數使用的估計方法 標準誤差0.81%
記數小時用稀疏矩陣節省空間
記數大時最小要12k的空間
適合非常大量用戶使用(set很大)

### bloom filter
節省90%空間但稍微不精確的set
已經大略知道size時可以調整空間降低誤差
https://krisives.github.io/bloom-calculator/
当布隆过滤器说某个值存在时，这个值可能不存在；当它说不存在时，那就肯定不存
在。
常用在
爬蟲系統:已经爬过的网页就可以不用爬了
noSQL: 当用户来查询某个 row 时，可以先通过内存中的布隆过滤器过滤掉大量不存在的
row 请求，然后再去磁盘进行查询。
垃圾郵件過濾

###　控制流量
系统要限定用户的某个行为在指定的时间里
只能允许发生 N 次，如何使用 Redis 的数据结构来实现这个限流的功能？

### 漏斗限流
漏斗的剩余空间就代表着当前行为可以持续进行的数量，漏嘴的流水速率代表着
系统允许该行为的最大频率

def make_space(self):
 now_ts = time.time()
 delta_ts = now_ts - self.leaking_ts # 距离上一次漏水过去了多久
 delta_quota = delta_ts * self.leaking_rate # 又可以腾出不少空间了
 if delta_quota < 1: # 腾的空间太少，那就等下次吧
 return
 self.left_quota += delta_quota # 增加剩余空间
 self.leaking_ts = now_ts # 记录漏水时间
 if self.left_quota > self.capacity: # 剩余空间不得高于容量
 self.left_quota = self.capacity

 redis-cell 可以用來實作

### GeoHash
要计算「附近的人」，也就是给定一个元素的坐标，然后计算这个坐标附近的其它元素
一般的方法都是通过矩形区域来限定元素的数量，然后对区域内的元素进行全量距离计算再排序
select id from positions where x0-r < x < x0+r and y0-r < y < y0+r
为了满足高性能的矩形区域算法，数据表需要在经纬度坐标加上双向复合索引 (x, y)，
这样可以最大优化查询性能。
但是数据库查询性能毕竟有限，如果「附近的人」查询请求非常多，在高并发场合，这
可能并不是一个很好的方案。
GeoHash 算法将二维的经纬度数据映射到一维的整数

在一个地图应用中，车的数据、餐馆的数据、人的数据可能会有百万千万条，如果使用
Redis 的 Geo 数据结构，它们将全部放在一个 zset 集合中。在 Redis 的集群环境中，集合
可能会从一个节点迁移到另一个节点，如果单个 key 的数据过大，会对集群的迁移工作造成
较大的影响，在集群环境中单个 key 对应的数据量不宜超过 1M，否则会导致集群迁移出现
卡顿现象，影响线上服务的正常运行。
所以，这里建议 Geo 的数据使用单独的 Redis 实例部署，不使用集群环境。
如果数据量过亿甚至更大，就需要对 Geo 数据进行拆分，按国家拆分、按省拆分，按
市拆分，在人口特大城市甚至可以按区拆分。这样就可以显著降低单个 zset 集合的大小。

### Scan
在平时线上 Redis 维护工作中，有时候需要从 Redis 实例成千上万的 key 中找出特定
前缀的 key 列表来手动处理数据，
keys 用来列出所有满足特定正则字符串规则的 key。
缺點
1.没有 offset、limit 参数，一次性吐出所有满足条件的 key，万一实例中有几百 w 个
key 满足条件，满屏的字符串刷的没有尽头
2.keys 算法是遍历算法，复杂度是 O(n)，如果实例中有千万级以上的 key，这个指令
就会导致 Redis 服务卡顿

Scan:
、复杂度虽然也是 O(n)，但是它是通过游标分步进行的，不会阻塞线程;
2、提供 limit 参数，可以控制每次返回结果的最大条数，limit 只是一个 hint，返回的
结果可多可少;
3、同 keys 一样，它也提供模式匹配功能;
4、服务器不需要为游标保存状态，游标的唯一状态就是 scan 返回给客户端的游标整数;
5、返回的结果可能会有重复，需要客户端去重复，这点非常重要;
6、遍历的过程中如果有数据修改，改动后的数据能不能遍历到是不确定的;
7、单次返回的结果是空的并不意味着遍历结束，而要看返回的游标值是否为零;

### 非阻塞IO
比如 read 方法要传递进去一个参数
n，表示读取这么多字节后再返回，如果没有读够线程就会卡在那里，直到新的数据到来或者
连接关闭了，read 方法才可以返回，线程才能继续处理。而 write 方法一般来说不会阻塞，除
非内核为套接字分配的写缓冲区已经满了，write 方法就会阻塞，直到缓存区中有空闲空间挪
出来了。
非阻塞 IO 在套接字对象上提供了一个选项 Non_Blocking，当这个选项打开时，读写方
法不会阻塞，而是能读多少读多少，能写多少写多少

### 事件轮询 (多路复用)
利用select

### 響應隊列
當客戶端向服務端發送一個請求時，服務端可以將該請求放到一個響應隊列中，然後立即回應客戶端。客戶端可以在後續的時間內從響應隊列中獲取結果。這樣可以實現異步處理，大大提高系統的吞吐量和可靠性。

### 定時任務
服务器处理要响应 IO 事件外，还要处理其它事情。比如定时任务就是非常重要的一件
事。如果线程阻塞在 select 系统调用上，定时任务将无法得到准时调度。那 Redis 是如何解
决这个问题的呢？
Redis 的定时任务会记录在一个称为最小堆的数据结构中。这个堆中，最快要执行的任
务排在堆的最上方。在每个循环周期，Redis 都会将最小堆里面已经到点的任务立即进行处
理。处理完毕后，将最快要执行的任务还需要的时间记录下来，这个时间就是 select 系统调
用的 timeout 参数。因为 Redis 知道未来 timeout 时间内，没有其它定时任务需要处理，所以
可以安心睡眠 timeout 的时间。

### persistance
快照:
全量备份, 二进制序列化

AOF:
是内存数据修改的指令记录文本

### 快照原理
Redis 使用操作系统的多进程 COW(Copy On Write) 机制来实现快照持久化
fork(多进程)
Redis 在持久化时会调用 glibc 的函数 fork 产生一个子进程，快照持久化完全交给子进
程来处理，父进程继续处理客户端请求。子进程刚刚产生时，它和父进程共享内存里面的代
码段和数据段。这时你可以将父子进程想像成一个连体婴儿，共享身体。这是 Linux 操作系
Redis 深度历险：核心原理与应用实践 | 钱文品 著
第 102 页 共 226 页
统的机制，为了节约内存资源，所以尽可能让它们共享起来。在进程分离的一瞬间，内存的
增长几乎没有明显变化。
子进程做数据持久化，它不会修改现有的内存数据结构，它只是对数据结构进行遍历读
取，然后序列化写到磁盘中。但是父进程不一样，它必须持续服务客户端请求，然后对内存
数据结构进行不间断的修改。
这个时候就会使用操作系统的 COW 机制来进行数据段页面的分离。数据段是由很多操
作系统的页面组合而成，当父进程对其中一个页面的数据进行修改时，会将被共享的页面复
制一份分离出来，然后对这个复制的页面进行修改。这时子进程相应的页面是没有变化的，
还是进程产生时那一瞬间的数据。

### AOF
Redis 会在收到客户端修改指令后，先进行参数校验，如果没问题，就立即将该指令文
本存储到 AOF 日志中，也就是先存到磁盘，然后再执行指令。这样即使遇到突发宕机，已
经存储到 AOF 日志的指令进行重放一下就可以恢复到宕机前的状态。
AOF 重写
Redis 提供了 bgrewriteaof 指令用于对 AOF 日志进行瘦身。其原理就是开辟一个子进
程对内存进行遍历转换成一系列 Redis 的操作指令，序列化到一个新的 AOF 日志文件中。
序列化完毕后再将操作期间发生的增量 AOF 日志追加到这个新的 AOF 日志文件中，追加
完毕后就立即替代旧的 AOF 日志文件了，瘦身工作就完成了。
fsync
AOF 日志是以文件的形式存在的，当程序对 AOF 日志文件进行写操作时，实际上是将
内容写到了内核为文件描述符分配的一个内存缓存中，然后内核会异步将脏数据刷回到磁盘
的。
这就意味着如果机器突然宕机，AOF 日志内容可能还没有来得及完全刷到磁盘中，这个
时候就会出现日志丢失。那该怎么办？
Linux 的 glibc 提供了 fsync(int fd)函数可以将指定文件的内容强制从内核缓存刷到磁
盘。只要 Redis 进程实时调用 fsync 函数就可以保证 aof 日志不丢失。但是 fsync 是一个
磁盘 IO 操作，它很慢！如果 Redis 执行一条指令就要 fsync 一次，那么 Redis 高性能的
地位就不保了。
所以在生产环境的服务器中，Redis 通常是每隔 1s 左右执行一次 fsync 操作，周期 1s
是可以配置的。这是在数据安全性和性能之间做了一个折中，在保持高性能的同时，尽可能
使得数据少丢失。
Redis 同样也提供了另外两种策略，一个是永不 fsync——让操作系统来决定合适同步磁盘，很不安全，另一个是来一个指令就 fsync 一次——非常慢。但是在生产环境基本不会使用，了解一下即可。

### Redis 4.0 混合持久化
将 rdb 文件的内容和增量的 AOF 日志文件存在一起。这里的 AOF 日志不再是全量的日志，而是自持久化开始到持久化结束的这段时间发生的增量 AOF 日志

### watch
watch 会在事务开始之前盯住 1 个或多个关键变量，当事务执行时，也就是服务器收到
了 exec 指令要顺序执行缓存的事务队列时，Redis 会检查关键变量自 watch 之后，是否被
修改了 (包括当前事务所在的客户端)。如果关键变量被人动过了，exec 指令就会返回 null
回复告知客户端事务执行失败，这个时候客户端一般会选择重试。

### pubsub
如果一个消费者都没有，那么消息直接丢弃。
如果 Redis 停机重启，PubSub 的消息是不会持久化的，毕竟 Redis 宕机就相当于一个消费者都没有，所有的消息直接被丢弃。
新的: Disque
redis5.0 開始可以用stream取代pubsub

### 節省內存的方法
32bit vs 64bit
Redis 如果使用 32bit 进行编译，内部所有数据结构所使用的指针空间占用会少一半，
如果你对 Redis 使用内存不超过 4G，可以考虑使用 32bit 进行编译，可以节约大量内存。

壓縮成ziplist

存储界限 当集合对象的元素不断增加，或者某个 value 值过大，这种小对象存储也会
被升级为标准结构。Redis 规定在小对象存储结构的限制条件如下：
hash-max-zipmap-entries 512 # hash 的元素个数超过 512 就必须用标准结构存储
hash-max-zipmap-value 64 # hash 的任意元素的 key/value 的长度超过 64 就必须用标准结构存储
list-max-ziplist-entries 512 # list 的元素个数超过 512 就必须用标准结构存储
list-max-ziplist-value 64 # list 的任意元素的长度超过 64 就必须用标准结构存储
zset-max-ziplist-entries 128 # zset 的元素个数超过 128 就必须用标准结构存储
zset-max-ziplist-value 64 # zset 的任意元素的长度超过 64 就必须用标准结构存储
set-max-intset-entries 512 # set 的整数元素个数超过 512 就必须用标准结构存储

内存回收机制
如果当前 Redis 内存有 10G，当你删除了 1GB 的 key 后，再去观察内存，你会发现
内存变化不会太大。原因是操作系统回收内存是以页为单位，如果这个页上只要有一个 key
还在使用，那么它就不能被回收。Redis 虽然删除了 1GB 的 key，但是这些 key 分散到了
很多页面中，每个页面都还有其它 key 存在，这就导致了内存不会立即被回收。

内存分配算法
目前 Redis 可以使用 jemalloc(facebook) 库来管理内
存，也可以切换到 tcmalloc(google)。因为 jemalloc 相比 tcmalloc 的性能要稍好一些，所以
Redis 默认使用了 jemalloc。

### CAP
C - Consistent ，一致性
A - Availability ，可用性
P - Partition tolerance ，分区容忍性

redis主從是async，因此僅保證最終一致性

### 增量同步
因为内存的 buffer 是有限的，所以 Redis 主库不能将所有的指令都记录在内存 buffer中。Redis 的复制内存 buffer 是一个定长的环形数组

### 快照同步
快照同步是一个非常耗费资源的操作，它首先需要在主库上进行一次 bgsave 将当前内存的数据全部快照到磁盘文件中，然后再将快照文件的内容全部传送到从节点。从节点将快照文件接受完毕后，立即执行一次全量加载，加载之前先要将当前内存的数据清空。加载完毕后通知主节点继续进行增量同步。
在整个快照同步进行的过程中，主节点的复制 buffer 还在不停的往前移动，如果快照同步的时间过长或者复制 buffer 太小，都会导致同步期间的增量指令在复制 buffer 中被覆盖，这样就会导致快照同步完成后无法进行增量复制，然后会再次发起快照同步，如此极有可能会陷入快照同步的死循环。所以务必配置一个合适的复制 buffer 大小参数，避免快照复制的死循环。

### 增加从节点
当从节点刚刚加入到集群时，它必须先要进行一次快照同步，同步完成后再继续进行增
量同步。

### 無盤複製
无盘复制是指主服务器直接通过套接字
将快照内容发送到从节点，生成快照是一个遍历的过程，主节点会一边遍历内存，一遍将序
列化的内容发送到从节点，从节点还是跟之前一样，先将接收到的内容存储到磁盘文件中，
再进行一次性加载。

### Sentinel
自動主從切換
它负责持续监控主从节点的健康，当主节点挂掉时，自动选择一个最优的从节点切换为
主节点。客户端来连接集群时，会首先连接 sentinel，通过 sentinel 来查询主节点的地址，
然后再去连接主节点进行数据交互。当主节点发生故障时，客户端会重新向 sentinel 要地
址，sentinel 会将最新的主节点地址告诉客户端。如此应用程序将无需重启即可自动完成节
点切换。

### 消息丢失
Redis 主从采用异步复制，意味着当主节点挂掉时，从节点可能没有收到全部的同步消
息，这部分未同步的消息就丢失了。如果主从延迟特别大，那么丢失的数据就可能会特别
多。Sentinel 无法保证消息完全不丢失，但是也尽可能保证消息少丢失。它有两个选项可以
限制主从延迟过大。
min-slaves-to-write 1
min-slaves-max-lag 10
第一个参数表示主节点必须至少有一个从节点在进行正常复制，否则就停止对外写服
务，丧失可用性。
何为正常复制，何为异常复制？这个就是由第二个参数控制的，它的单位是秒，表示如
果 10s 没有收到从节点的反馈，就意味着从节点同步不正常，要么网络断开了，要么一直没
有给反馈。

### codis
Codis 使用 Go 语言开发，它是一个代理中间件，它和 Redis 一样也使用 Redis 协议
对外提供服务，当客户端向 Codis 发送指令时，Codis 负责将指令转发到后面的 Redis 实例
来执行，并将返回结果再转回给客户端。
Codis 将所有的 key 默认划分为 1024 个槽位(slot)
如果 Codis 的槽位映射关系只存储在内存里，那么不同的 Codis 实例之间的槽位关系
就无法得到同步。所以 Codis 还需要一个分布式配置存储数据库专门用来持久化槽位关系。
Codis 开始使用 ZooKeeper，后来连 etcd 也一块支持了。

#### 扩容
刚开始 Codis 后端只有一个 Redis 实例，1024 个槽位全部指向同一个 Redis。然后一
个 Redis 实例内存不够了，所以又加了一个 Redis 实例。这时候需要对槽位关系进行调整，
将一半的槽位划分到新的节点。这意味着需要对这一半的槽位对应的所有 key 进行迁移，迁
移到新的 Redis 实例。
那 Codis 如何找到槽位对应的所有 key 呢？
Codis 对 Redis 进行了改造，增加了 SLOTSSCAN 指令，可以遍历指定 slot 下所有的
key。Codis 通过 SLOTSSCAN 扫描出待迁移槽位的所有的 key，然后挨个迁移每个 key 到
新的 Redis 节点。
在迁移过程中，Codis 还是会接收到新的请求打在当前正在迁移的槽位上，因为当前槽
位的数据同时存在于新旧两个槽位中，Codis 如何判断该将请求转发到后面的哪个具体实例
呢？
Codis 无法判定迁移过程中的 key 究竟在哪个实例中，所以它采用了另一种完全不同的
思路。当 Codis 接收到位于正在迁移槽位中的 key 后，会立即强制对当前的单个 key 进行
迁移，迁移完成后，再将请求转发到新的 Redis 实例。

#### 自动均衡

#### Codis 的代价
key的位置分散因此不支持事務
zookeeper運維
多了proxy節點，網路的開銷

#### Codis 的优点
Codis 在设计上相比 Redis Cluster 官方集群方案要简单很多，因为它将分布式的问题交
给了第三方 zk/etcd 去负责，自己就省去了复杂的分布式一致性代码的编写维护工作。而
Redis Cluster 的内部实现非常复杂，它为了实现去中心化，混合使用了复杂的 Raft 和
Gossip 协议，还有大量的需要调优的配置参数，当集群出现故障时，维护人员往往不知道从
何处着手。

### Cluster
去中心化, 由三個節點組成

## Stream
consumer是競爭關係，且有透過ack實作at least one

High availability是透過主從複製，也就是cluster環境

## Redlock
如果你很在乎高可用性，希望挂了一台 redis 完全不受影响，那就应该考虑 redlock。不
过代价也是有的，需要更多的 redis 实例，性能也下降了，代码上还需要引入额外的
library，运维上也需要特殊对待，这些都是需要考虑的成本，使用前请再三斟酌。

## 過期策略
定時刪除 ＆ 惰性刪除
所以业务开发人员一定要注意过期时间，如果有大批量的 key 过期，要给过期时间设置
一个随机范围，而不能全部在同一时间过期。避免卡頓

## LRU
1. noevition:can delete & read, cannot write
2. volatile-lru: 尝试淘汰设置了过期时间的 key，最少使用的 key 优先被淘汰
3. volatile-ttl: 跟上面一样，除了淘汰的策略不是 LRU，而是 key 的剩余寿命 ttl 值，ttl越小越优先被淘汰。
4. volatile-random 跟上面一样，不过淘汰的 key 是过期 key 集合中随机的 key。
5. allkeys-lru 区别于 volatile-lru，这个策略要淘汰的 key 对象是全体的 key 集合，而不只是过期的 key 集合。这意味着没有设置过期时间的 key 也会被淘汰。
6. allkeys-random 跟上面一样，不过淘汰的策略是随机的 key。
7. volatile-xxx 策略只会针对带过期时间的 key 进行淘汰，allkeys-xxx 策略会对所有的key 进行淘汰。如果你只是拿 Redis 做缓存，那应该使用 allkeys-xxx，客户端写缓存时不必携带过期时间。如果你还想同时使用 Redis 的持久化功能，那就使用 volatile-xxx策略，这样可以保留没有设置过期时间的 key，它们是永久的 key 不会被 LRU 算法淘汰。

redis使用近似LRU，原因是LRU需要過多的記憶體
Redis 为实现近似 LRU 算法，它给每个 key 增加了一个额外的小字
段，这个字段的长度是 24 个 bit，也就是最后一次被访问的时间戳。
LRU 淘汰不一样，它的处理
方式只有懒惰处理。当 Redis 执行写操作时，发现内存超出 maxmemory，就会执行一次
LRU 淘汰算法。这个算法也很简单，就是随机采样出 5(可以配置) 个 key，然后淘汰掉最
旧的 key，如果淘汰后内存还是超出 maxmemory，那就继续随机采样淘汰，直到内存低于
maxmemory 为止。
如何采样就是看 maxmemory-policy 的配置，如果是 allkeys 就是从所有的 key 字典中
随机，如果是 volatile 就从带过期时间的 key 字典中随机。每次采样多少个 key 看的是
maxmemory_samples 的配置，默认为 5。
redis增加了淘汰池
淘汰池是一个数组，它的大小是 maxmemory_samples，在每一次淘汰循环中，新随机出
来的 key 列表会和淘汰池中的 key 列表进行融合，淘汰掉最旧的一个 key 之后，保留剩余
较旧的 key 列表放入淘汰池中留待下一个循环

## String deep
SDS simple dynamic string

### embstr vs raw
<=44 : emb: SDS及object連續
/>44 : raw： 不連續

1M前 加倍擴容
之後都1M

## Dict deep
hash func: siphash

## hash attack
如果 hash 函数存在偏向性，黑客就可能利用这种偏向性对服务器进行攻击。存在偏向
性的 hash 函数在特定模式下的输入会导致 hash 第二维链表长度极为不均匀，甚至所有的
元素都集中到个别链表中，直接导致查找效率急剧下降，从 O(1)退化到 O(n)。有限的服务器
计算能力将会被 hashtable 的查找效率彻底拖垮。这就是所谓 hash 攻击。

## zip list

## IntSet

## quickList
zipList + linkedList
LZF算法壓縮
zipList length = 8k bytes
quicklist 默认的压缩深度是 0，也就是不压缩。压缩的实际深度由配置参数 listcompress-depth 决定。为了支持快速的 push/pop 操作，quicklist 的首尾两个 ziplist 不压
缩，此时深度就是 1。如果深度为 2，就表示 quicklist 的首尾第一个 ziplist 以及首尾第二
个 ziplist 都不压缩。
reference: https://matt.sh/redis-quicklist

## jumpList
zset = hash + skipList

## listpack
用來取代ziplist

## Rax
rax: 按key排序
zset: 按score排序
被用在stream儲存消息隊列

# extend
最近看到一篇质量很高的博客文章 《Redis: under the hood》
这篇博文是老外写的，值得有英语能力的同学学习。这篇文章会带你一步步学会如何阅
读 Redis 的源码，有一定的难度，不过整体来说还算通俗易懂。文章中 vm 相关代码片段不
用去理会，这个功能很早就被 Redis 官方废弃了。

