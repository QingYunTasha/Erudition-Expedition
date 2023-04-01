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