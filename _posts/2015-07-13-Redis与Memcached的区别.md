---
layout: post
title:  Redis与Memcached的区别
---

##持久化
---
###RDB(快照)
####优点
* RDB会把某一时间点的在redis的数据压缩到单个文件中，在实际中，可以获取过去24小时中每个小时的RDB文件，也可以每天保存。灾难恢复时很容易的使用不同版本恢复
* RDB非常适合容灾，单个压缩文件可以传到远程数据中心
* RDB最大化了Redis性能，Redis主线程fork子进程然后其它事情由子进程处理，主进程不处理磁盘IO操作
* RDB在重启大数据集时，速度更快

####缺点
* 数据丢失：如果希望在redis挂掉时尽可能少的丢数据，RDB是不合适的。可以配置不同的保存点（至少5分钟且有100次数据写入，可以有多个保存点），每5分钟或更多时间会有一次快照，但是如果redis非正常原因停止工作，会丢失最近的数据。
* 时间消耗：Redis为了持久化到磁盘，需要经常fork子进程。如果数据很大，fork会很消耗时间，可能会导致redis停止服务几微秒，如果数据非常大而且cpu不给力可能会有1秒。虽然AOF也需要fork，但是你可以调整重写Log的频率，且在执行期间不带有额外开销

###AOF(记录每一次写操作)
####优点：
* 使用AOF，Redis更持久：可以使用不同的文件同步策略，不同步、每秒同步、每次请求同步。默认策略每秒同步性能也很好（fsync使用一个后台线程实现，而没有fsync执行时，主线程尽可能执行写操作），最多丢一秒的数据。
* AOF是只追加型的日志，所以断电时，不会丢数据。就算是日志结尾写入一半（磁盘满），仍然可以使用redis-check-aof轻松修复。
* 当AOF太大时Redis可以在后台自动重写，重写后的新 AOF 文件包含了恢复当前数据集所需的最小命令集合。。重写是完全安全的，因为 Redis 在创建新 AOF 文件的过程中，会继续将命令追加到现有的 AOF 文件里面，即使重写过程中发生停机，现有的 AOF 文件也不会丢失。 而一旦新 AOF 文件创建完毕，Redis 就会从旧 AOF 文件切换到新 AOF 文件，并开始对新 * AOF 文件进行追加操作。
* AOF 文件有序地保存了对数据库执行的所有写入操作， 这些写入操作以 Redis 协议的格式保存， 因此 AOF 文件的内容非常容易被人读懂，对文件进行分析（parse）也很轻松。 导出（export） AOF 文件也非常简单：举个例子， 如果你不小心执行了 FLUSHALL 命令， 但只要 AOF 文件未被重写，那么只要停止服务器， 移除 AOF 文件末尾的 FLUSHALL 命令，并重启 Redis ，就可以将数据集恢复到 FLUSHALL 执行之前的状态。

####缺点：
* 对于相同的数据集来说，AOF 文件的体积通常要大于 RDB 文件的体积。
* 根据所使用的 fsync 策略，AOF 的速度可能会慢于 RDB 。 在一般情况下， 每秒 fsync 的性能依然非常高， 而关闭 fsync 可以让 AOF 的速度和 RDB 一样快， 即使在高负荷之下也是如此。 不过在处理巨大的写入载入时，RDB 可以提供更有保证的最大延迟时间（latency）
可以禁用持久化
结合使用
如何选择
一般来说， 如果想达到足以媲美 PostgreSQL 的数据安全性， 你应该同时使用两种持久化功能。
如果你非常关心你的数据， 但仍然可以承受数分钟以内的数据丢失， 那么你可以只使用 RDB 持久化。
有很多用户都只使用 AOF 持久化， 但我们并不推荐这种方式： 因为定时生成 RDB 快照（snapshot）非常便于进行数据库备份， 并且 RDB 恢复数据集的速度也要比 AOF 恢复的速度要快。
当 Redis 启动时， 如果 RDB 持久化和 AOF 持久化都被打开了， 那么程序会优先使用 AOF 文件来恢复数据集， 因为 AOF 文件所保存的数据通常是最完整的。

##数据结构
---
Memcache: kv

Redis: kv, hashes, lists, sets, sorted sets

##过期
---
####Memcache
* 访问Key的时候，发现超时，标识删除为1
* 不会真正删除分配的内存，覆盖新的key

####Redis
* 积极方式 当访问key的时候，会发现是否超时
* 消积方式 定期随机检测若干个key，如果过期则删除。每秒执行10次如下操作：在设置过期时间的key中随机检测20个key，删除过期的key，如果超过25%的key被删除，再执行一遍

##LRU
---
###设置最大内存 
* 可以在配置文件中设置 也可以用命令设置
* 设置0表示没有内存限制
* 到达指定内存时，会有不同的处理行为，即策略。如报错，清理旧数据等。

###回收策略
* noeviction 不回收，直接报错
* allkeys-lru 优先回收最近很少使用的key 为新数据增加空间
* volatile-lru 优先回收最近很少使用的【且设置超时的】key
* allkeys-random 所有key中随机回收
* volatile-random 在设置超时的key中随机回收
* volatile-ttl 回收最近很少使用的【且设置超时的】key 且优先回收存活时间短的

###Redis实现
* allkeys和volatile分别在两个table中
* random则直接随机取出若干个
* LRU的实现是近似LRU。之前版本实现：随机找三条记录出来，比较哪条空闲时间最长就删哪条，然后再随机三条出来，一直删到内存足够放下新记录为止。3.0版本实现：现在每次随机五条记录出来，插入到一个长度为十六的按空闲时间排序的队列里，然后把排头的那条删掉，然后再找五条出来，继续尝试插入队列。

###Memcached实现
* 最大可设置2G内存
* LRU列队，双向链表

##参考资料
---
* [Redis Persistence](http://redis.io/topics/persistence)
* [Using Redis as an LRU cache](http://redis.io/topics/lru-cache)
* [LRU算法的实现，简单粗暴的Redis与中规中矩的Memcached](http://www.yidianzixun.com/08lZsltN?s=undefined)
* [memcached源码分析-----LRU队列与item结构体](http://blog.csdn.net/luotuo44/article/details/42869325)
* [Using hashes to abstract a very memory efficient plain key-value store on top of Redis](http://redis.io/topics/memory-optimization)
* [节约内存：Instagram的Redis实践](http://blog.nosqlfan.com/html/3379.html)