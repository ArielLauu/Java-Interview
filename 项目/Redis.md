### Redis

---

[TOC]

#### 为什么要用Redis

从“高性能”和“高并发”这两点来看待这个问题

**1. 高性能**

假如用户第一次访问数据库中的某些数据的话，这个过程是比较慢，毕竟是从硬盘中读取的。但是，如果说，用户访问的数据属于高频数据并且不会经常改变的话，那么我们就可以很放心地将该用户访问的数据存在缓存中

**这样有什么好处呢？** 那就是保证用户下一次再访问这些数据的时候就可以直接从缓存中获取了。操作缓存就是直接操作内存，所以速度相当快。

**2. 高并发**

一般像 MySQL 这类的数据库的 QPS 大概都在 1w 左右（4 核 8g） ，但是使用 Redis 缓存之后很容易达到 10w+，甚至最高能达到 30w+（就单机 redis 的情况，redis 集群的话会更高）。

所以，直接操作缓存能够承受的数据库请求数量是远远大于直接访问数据库的，所以我们可以考虑把数据库中的部分数据转移到缓存中去，这样用户的一部分请求会直接到缓存这里而不用经过数据库。进而，我们也就提高的系统整体的并发。

---

#### Redis数据类型

一共有五种，string / list / hash / set / sorted set

**1. string**

string 数据结构是简单的 key-value 类型

**常用命令:** `set,get,strlen,exists,dect,incr,setex` 等等。

**应用场景** ：一般常用在需要计数的场景，比如用户的访问次数、热点文章的点赞转发数量等等。

**2. list**

Redis 的 list 的实现为一个 **双向链表**，即可以支持反向查找和遍历，更方便操作，不过带来了部分额外的内存开销。

- **常用命令:** `rpush,lpop,lpush,rpop,lrange、llen` 等。

- **应用场景:** 发布与订阅或者说消息队列、慢查询。

**3. hash**

hash 类似于 JDK1.8 前的 HashMap，内部实现也差不多(数组 + 链表)。不过，Redis 的 hash 做了更多优化。另外，hash 是一个 string 类型的 field 和 value 的映射表，**特别适合用于存储对象**

- **常用命令：** `hset,hmset,hexists,hget,hgetall,hkeys,hvals` 等。

- **应用场景:** 系统中对象数据的存储。

**4. set**

set 类似于 Java 中的 `HashSet` 。Redis 中的 set 类型是一种无序集合，没有重复数据。

 set 提供了判断某个成员是否在一个 set 集合内的重要接口，这个也是 list 所不能提供的。

可以基于 set 轻易实现交集、并集、差集的操作。比如：你可以将一个用户所有的关注人存在一个集合中，将其所有粉丝存在一个集合。Redis 可以非常方便的实现如共同关注、共同粉丝、共同喜好等功能。这个过程也就是求交集的过程。

- **常用命令：** `sadd,spop,smembers,sismember,scard,sinterstore,sunion` 等。

- **应用场景:** 需要存放的数据不能重复以及需要获取多个数据源交集和并集等场景

**5. sorted set**

 和 set 相比，sorted set 增加了一个权重参数 score，使得集合中的元素能够按 score 进行有序排列，还可以通过 score 的范围来获取元素的列表。有点像是 Java 中 HashMap 和 TreeSet 的结合体。

- **常用命令：** `zadd,zcard,zscore,zrange,zrevrange,zrem` 等。

- **应用场景：** 需要对数据根据某个权重进行排序的场景。比如在直播系统中，实时排行信息包含直播间在线用户列表，各种礼物排行榜，弹幕消息（可以理解为按消息维度的消息排行榜）等信息。

---

#### Redis处理过期数据

Redis 通过一个叫做**过期字典**（可以看作是hash表）来保存数据过期的时间。过期字典的键指向Redis数据库中的某个key(键)，过期字典的值是一个long long类型的整数，这个整数保存了key所指向的数据库键的过期时间

常用的过期数据的删除策略：

1. **惰性删除** ：只会在取出key的时候才对数据进行过期检查。这样对CPU最友好，但是可能会造成太多过期 key 没有被删除。
2. **定期删除** ： 每隔一段时间抽取一批 key 执行删除过期key操作。并且，Redis 底层会通过限制删除操作执行的时长和频率来减少删除操作对CPU时间的影响。

---

#### Redis内存淘汰机制

1. **volatile-lru（least recently used）**：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
2. **volatile-ttl**：从已设置过期时间的数据集中，挑选将要过期的数据淘汰
3. **volatile-random**：从已设置过期时间的数据集中，任意选择数据淘汰
4. **allkeys-lru（least recently used）**：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 key（这个是最常用的）
5. **allkeys-random**：从数据集（server.db[i].dict）中任意选择数据淘汰
6. **no-eviction**：禁止驱逐数据，也就是说当内存不足以容纳新写入数据时，新写入操作会报错。这个应该没人使用吧！

4.0 版本后增加以下两种：

1. **volatile-lfu（least frequently used）**：从已设置过期时间的数据集中，挑选最不经常使用的数据淘汰
2. **allkeys-lfu（least frequently used）**：当内存不足以容纳新写入数据时，在键空间中，移除最不经常使用的 key

---

#### Redis持久化机制

**持久化作用**：持久化数据也就是将内存中的数据写入到硬盘里面，大部分原因是为了之后重用数据（比如重启机器、机器故障之后恢复数据），或者是为了防止系统故障而将数据备份到一个远程位置。

**1. 快照（snapshotting）持久化（RDB）**

Redis 可以通过创建快照来获得存储在内存里面的数据在某个时间点上的副本。

Redis 创建快照之后，可以**对快照进行备份**，可以将快照复制到其他服务器从而创建具有相同数据的服务器副本（Redis 主从结构，主要用来提高 Redis 性能），还可以将快照留在原地以便重启服务器的时候使用。

快照持久化是 Redis 默认采用的持久化方式，在 Redis.conf 配置文件中默认有此下配置：

```conf
save 900 1     #在900秒(15分钟)之后，如果至少有1个key发生变化，Redis就会自动触发BGSAVE命令创建快照。

save 300 10    #在300秒(5分钟)之后，如果至少有10个key发生变化，Redis就会自动触发BGSAVE命令创建快照。

save 60 10000  #在60秒(1分钟)之后，如果至少有10000个key发生变化，Redis就会自动触发BGSAVE命令创建快照。
```

**2. AOF（append-only file）持久化**

与快照持久化相比，AOF 持久化 的实时性更好，因此已成为主流的持久化方案。默认情况下 Redis 没有开启 AOF（append only file）方式的持久化，可以通过 appendonly 参数开启：

```conf
appendonly yesCopy to clipboardErrorCopied
```

开启 AOF 持久化后每执行一条会更改 Redis 中的数据的命令，Redis 就会将该命令写入硬盘中的 AOF 文件。AOF 文件的保存位置和 RDB 文件的位置相同，都是通过 dir 参数设置的，默认的文件名是 appendonly.aof。

在 Redis 的配置文件中存在三种不同的 AOF 持久化方式，它们分别是：

```java
appendfsync always    #每次有数据修改发生时都会写入AOF文件,这样会严重降低Redis的速度
appendfsync everysec  #每秒钟同步一次，显示地将多个写命令同步到硬盘
appendfsync no        #让操作系统决定何时进行同步Copy to clipboardErrorCopied
```

为了兼顾数据和写入性能，用户可以考虑 appendfsync everysec 选项 ，让 Redis 每秒同步一次 AOF 文件，Redis 性能几乎没受到任何影响。而且这样即使出现系统崩溃，用户最多只会丢失一秒之内产生的数据。当硬盘忙于执行写入操作的时候，Redis 还会优雅的放慢自己的速度以便适应硬盘的最大写入速度。

---

#### 缓存穿透

缓存穿透说简单点就是大量请求的 key 根本不存在于缓存中，导致请求直接到了数据库上，根本没有经过缓存这一层。举个例子：某个黑客故意制造我们缓存中不存在的 key 发起大量请求，导致大量请求落到数据库。

**解决办法**

**1. 缓存无效 key**

这种方式可以解决请求的 key 变化不频繁的情况，如果黑客恶意攻击，每次构建不同的请求 key，会导致 Redis 中缓存大量无效的 key 。

**2. 布隆过滤器**

通过它我们可以非常方便地判断一个给定数据是否存在于海量数据中

**具体流程**：把所有可能存在的请求的值都存放在布隆过滤器中，当用户请求过来，先判断用户发来的请求的值是否存在于布隆过滤器中。不存在的话，直接返回请求参数错误信息给客户端，存在的话才会走下面的流程。

我们先来看一下，**当一个元素加入布隆过滤器中的时候，会进行哪些操作：**

1. 使用布隆过滤器中的哈希函数对元素值进行计算，得到哈希值（有几个哈希函数得到几个哈希值）。
2. 根据得到的哈希值，在位数组中把对应下标的值置为 1。

我们再来看一下，**当我们需要判断一个元素是否存在于布隆过滤器的时候，会进行哪些操作：**

1. 对给定元素再次进行相同的哈希计算；
2. 得到值之后判断位数组中的每个元素是否都为 1，如果值都为 1，那么说明这个值在布隆过滤器中，如果存在一个值不为 1，说明该元素不在布隆过滤器中。

然后，一定会出现这样一种情况：**不同的字符串可能哈希出来的位置相同。** 

---

#### 缓存雪崩

解释：缓存在同一时间大面积的失效，后面的请求都直接落到了数据库上，造成数据库短时间内承受大量请求

系统的**缓存模块出了问题**比如宕机导致不可用。造成系统的所有访问，都要走数据库。或者有一些被大量访问数据**（热点缓存）在某一时刻大面积失效**，导致对应的请求直接落到了数据库上。

**解决办法**

**针对 Redis 服务不可用的情况：**

1. 采用 Redis 集群，避免单机出现问题整个缓存服务都没办法使用。
2. 限流，避免同时处理大量的请求。

**针对热点缓存失效的情况：**

1. 设置不同的失效时间比如随机设置缓存的失效时间。
2. 缓存永不失效。

---

#### Redis 集群架构

**1. 主从模式**

<img src="https://typora-image-elias.oss-cn-hangzhou.aliyuncs.com/uPic/image-20210308215155824.png" alt="image-20210308215155824" style="zoom:50%;" />

master用来支持数据的写入和读取操作，而slave支持读取及master的数据同步，在整个架构里，master和slave实例里的数据完全一致

**问题:** 主节点因为故障下线后，需要手动更改客户端配置重新连接，这种模式并不能保证服务的高可用

**2. 哨兵模式**

<img src="https://typora-image-elias.oss-cn-hangzhou.aliyuncs.com/uPic/image-20210308215459652.png" alt="image-20210308215459652" style="zoom:50%;" />

增加了独立进程（即哨兵）来监控集群中的一举一动。客户端在连接集群时，首先连接哨兵，通过哨兵查询主节点的地址，然后再去连接主节点进行数据交互。

如果master异常 (哨兵每秒向集群中的master、slave以及其他哨兵发送一个PING命令，超时会被标记**主观下线**，其他哨兵也会来确认主观下线状态，满足数量就标记为下线)，则会进行master-slave切换，将最优的一个slave切换为主节点。

**问题: ** 随着业务的逐渐增长，最终还是需要通过水平扩容方式来解决。而水平扩容涉及到数据的迁移，且迁移过程中又要保证服务的可用性。

**3. Cluster模式**

<img src="https://typora-image-elias.oss-cn-hangzhou.aliyuncs.com/uPic/image-20210308220024434.png" alt="image-20210308220024434" style="zoom:50%;" />

1. redis cluster模式采用了无中心节点的方式来实现，每个主节点都会与其它主节点保持连接。节点间通过gossip协议交换彼此的信息，同时每个主节点又有一个或多个从节点；
2. 客户端连接集群时，直接与redis集群的每个主节点连接，根据hash算法取模将key存储在不同的哈希槽上；
3. 在集群中采用数据分片的方式，将redis集群分为16384个哈希槽。这些哈希槽分别存储于三个主节点中

4. 每个节点会保存一份数据分布表，节点会将自己的slot信息发送给其他节点，节点间不停的传递数据分布表

5. 客户端连接集群时，通过集群中某个节点地址进行连接。客户端尝试向这个节点执行命令时，比如获取某个key值，如果key所在的slot刚好在该节点上，则能够直接执行成功。如果slot不在该节点，则节点会返回MOVED错误，同时把该slot对应的节点告诉客户端，客户端可以去该节点执行命令。

**解决扩容问题:** 

当集群中加入新节点时，会与集群中的某个节点进行握手，该节点会把集群内的其它节点信息通过gossip协议(类似一传10，10传百）发送给新节点，新节点与这些节点完成握手后加入到集群中 ,然后集群中的节点会各取一部分哈希槽分配给新节点

**4. 主从复制**

- **全量同步**

1. 当从节点启动时，会向主节点发送SYNC命令；
2. 主节点接收到SYNC命令后，开始在后台执行保存快照的命令生成RDB文件，并使用缓冲区记录此后执行的所有写命令；
3. 主节点快照完成后，将快照文件和所有缓存命令发送给集群内的从节点，并在发送期间继续记录被执行的写命令；
4. 主节点快照发送完毕后开始向从节点发送缓冲区中的写命令；
5. 从节点载入快照文件后，开始接收命令请求，执行接收到的主节点缓冲区的写命令。

- **增量同步**

主从复制中因网络等原因造成数据丢失场景，当从节点再次连上主节点。如果条件允许，主节点会补发丢失数据给从节点。因为补发的数据远远小于全量数据，可以有效避免全量复制的过高开销。

[Redis 集群](https://juejin.cn/post/6869619522559541255)