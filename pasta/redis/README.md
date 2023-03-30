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