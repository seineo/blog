---
title: Redis的数据淘汰策略
date: 2024-07-13 17:08:43
tags:
---

>   P.S. 本文主要参考的是Redis 2.6版本的源码，但也在最后提及了Redis 7.0.3版本达到内存限制时的数据淘汰策略。

Redis有两种情况需要进行数据淘汰：

1.  **数据过期**：用户可以为每个键设置一个过期时间，当键达到指定的过期时间时，Redis会自动将其删除，以确保数据的有效性。
2.  **内存限制**：当Redis的内存使用达到预设的限制时，系统会根据配置的策略进行数据淘汰。这些策略帮助Redis在内存有限的情况下，尽可能保留重要数据，同时删除不常使用或过期的数据，保证系统的稳定运行。

## 数据过期

Redis处理过期键是 **惰性删除+定时删除** 的组合，通过结合这两种删除策略，Redis既能在低负载情况下通过惰性删除减少不必要的资源消耗，又能通过定时删除确保过期键不会长期占用内存，从而优化系统的整体性能和内存使用。

### 惰性删除

在数据库读写键时，Redis会调用`expireIfNeeded` 函数检查键是否过期，是则删除。然后才会在数据字典中查找键，记录信息，如果存在则更新LRU。

`expireIfNeeded` 函数会从过期字典中获取过期时间， 没有过期时间直接返回，否则

-    如果本节点是master节点，则判断当前时间是否大于过期时间，是则传播过期命令，然后从数据字典删除数据。
-    如果本节点是slave节点，则直接返回，因为slave节点的过期是由主节点通过发送 DEL命令来删除的，不必自主删除。

其中“传播过期命令”是由`propagateExpire`函数完成的，是当一个键在master节点中过期时，传播一个`DEL`命令到所有slave节点，并添加到AOF文件，保证数据一致性。

### 定时删除

Redis的cron是默认每10ms调用一次，其中定时主动删除过期key的函数为`activeExpireCycle` 。

```Plain
/* Try to expire a few timed out keys. The algorithm used is adaptive and
  will use few CPU cycles if there are few expiring keys, otherwise
  it will get more aggressive to avoid that too much memory is used by
  keys that can be removed from the keyspace. */
```

从注释可以看出，Redis的定期删除是一种自适应的算法，如果过期的键比较少那么占用的CPU时间就短，过期键比较多就会为了避免内存占用过高而用更长的CPU时间去清除数据。

步骤如下：

1.  遍历处理每个db，获取当前db的expires过期字典。
2.  如果expires键数量不到expires分配大小的1%，则暂时不处理。
3.  否则每次随机抽取`REDIS_EXPIRELOOKUPS_PER_CRON`（默认为10）个键，如果过期则删除，并将expired计数加一。如果这次expired的个数超过`REDIS_EXPIRELOOKUPS_PER_CRON`的25%，则说明过期键比较多，继续这一个循环删除。
4.  即使过期键很多，也不能过多占用cpu时间，Redis设置了该cron的最长占用时间，每16次循环判断一下cron执行是否超时，是则结束。

### Q&A

Q：定时删除是多线程执行的吗？那这样不是得加锁？

A：定时删除并不是多线程执行的，它属于Redis的一个定时任务，定时任务会作为一种时间事件（TimeEvent）通过`aeCreateTimeEvent`函数注册到Redis使用的Reactor框架中去，然后利用I0多路复用，单线程地完成这些任务，十分巧妙地避免了多线程导致的锁竞争问题。

## 内存限制

当我们在Redis配置文件中设置`maxmemory <byte>` 后，Redis会定时调用`freeMemoryIfNeeded` 函数，它会检查Redis当前的内存占用，如果超过了内存限制，则根据配置的淘汰策略不断挑选出最符合要求的键并删除掉，直到内存占用低于指定的最大内存限制。

Redis 2.6版本支持LRU、TTL、random以及noeviction这四种淘汰策略，随着Redis版本的迭代，也支持了越来越多的淘汰策略，比如我现在在使用的Redis 7.0.3版本，根据`Redis.conf`文件我们可以看到有8种淘汰策略：

```text
volatile-lru -> Evict using approximated LRU, only keys with an expire set.
allkeys-lru -> Evict any key using approximated LRU.
volatile-lfu -> Evict using approximated LFU, only keys with an expire set.
allkeys-lfu -> Evict any key using approximated LFU.
volatile-random -> Remove a random key having an expire set.
allkeys-random -> Remove a random key, any key.
volatile-ttl -> Remove the key with the nearest expire time (minor TTL)
noeviction -> Don't evict anything, just return an error on write operations.
```

`noeviction`是默认模式，不驱逐数据，但不让写入数据了，内存满时再写入会报错。

这里我们着重了解一下常用的LRU、LFU策略，其实Redis的实现并不是严谨地驱逐最近未使用或最近使用次数最少的键值数据，这样实现会比较复杂（比如LRU需要维护链表，这增加了内存占用与数据移动），而是采用有些随机的算法并支持用户调参来达到近似的结果。

### LRU

Redis会随机选取5个键，选取最久未使用的键淘汰掉。采样个数是用户可以调整的，默认的5个已经可以达到不错的效果，10个会更接近真实的LRU算法但是更占用CPU资源，3个会更快但是效果更差。

### LFU

Redis对每一个key维护一个8 bit的计数器，那么最大的使用次数只能表示到255。Redis并不是每次访问都会将计数器加一，基本原则是**访问次数越多，计数器加一的概率越低**。具体的计算方式如下：

```Plain
Given the value of the old counter, when a key is accessed, the counter is incremented in this way:

1. A random number R between 0 and 1 is extracted.
2. A probability P is calculated as 1/(old_value*lfu_log_factor+1).
3. The counter is incremented only if R < P.

The default lfu-log-factor is 10. This is a table of how the frequency counter changes with a different number of accesses with different logarithmic factors:

+--------+------------+------------+------------+------------+------------+
| factor | 100 hits   | 1000 hits  | 100K hits  | 1M hits    | 10M hits   |
+--------+------------+------------+------------+------------+------------+
| 0      | 104        | 255        | 255        | 255        | 255        |
+--------+------------+------------+------------+------------+------------+
| 1      | 18         | 49         | 255        | 255        | 255        |
+--------+------------+------------+------------+------------+------------+
| 10     | 10         | 18         | 142        | 255        | 255        |
+--------+------------+------------+------------+------------+------------+
| 100    | 8          | 11         | 49         | 143        | 255        |
+--------+------------+------------+------------+------------+------------+
```

如果只是不断向上计数，哪怕我们用64 bit来存储计数最后都会用完，大家都达到最大值时这个算法也就没有意义了，因此我们需要定期地让计数值衰减。Redis默认的衰减时间为一分钟，每次访问键时会判断是否上次更新时间已经过了一分钟，如果是，则根据计数值大小来衰减：

1.  如果计数值大于10，计数值减半。
2.  如果计数值小于10，计数值减一。

## 总结

无论是将定时任务作为时间事件，从而可以与用户的读写事件一起用单线程版的Reactor模式处理，避免了多线程的锁竞争问题，还是为了减小内存占用与实现复杂度，使用随机采样的算法达到近似LRU和LFU的算法效果，Redis数据淘汰策略的简洁、巧妙与高效都让我受益匪浅。
