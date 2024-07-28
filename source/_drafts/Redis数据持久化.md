---
title: Redis数据持久化
date: 2024-07-27 21:25:10
tags:
---

Redis是一款基于内存存储的键值数据库，那么默认情况下，当Redis服务器重启或者宕机后数据就会丢失。在大多业务场景下，我们都不希望数据丢失，至少不要全部丢失，因此Redis提供了数据库持久化的特性，允许数据存储到磁盘以应对服务器宕机或重启的情况。

Redis有两种持久化方案：

-   AOF：
-   RDB：

接下来会结合Redis 2.6版本源码来学习Redis是如何实现数据持久化的，这两个方案留存至今，只不过实现细节会有所变化。**AOF与RDB的大致流程相似，因此本文主要会分析AOF的实现流程**，并在最后讨论AOF与RDB两种方案的区别以及如何选择。

## AOF实现

### 主流程

服务启动会读取config文件，根据appendonly是否设置，将aof_state设置为REDIS_AOF_ON或 REDIS_AOF_OFF。

rewriteAppendOnlyFileBackground函数会fork一个子进程，在子进程中调用rewriteAppendOnlyFile函数重写命令到临时的AOF文件（temp-rewriteaof-bg-{getpid()}.aof）中，父进程负责处理错误，如果创建子进程成功则更新服务器关于AOF的状态（如aof_child_pid，不为-1了说明后台正在执行aof）。

rewriteAppendOnlyFile函数会尝试将每个数据库的键值SET命令保存在自己的临时文件中（temp-rewriteaof-{getpid()}.aof）。全部成功写完后，fflush刷新用户缓冲区写到内核缓冲区，fsync刷新内核缓冲区确保数据都写入了文件，然后通过更名，用重写后的新 AOF 文件（这里是temp-rewriteaof-pid.aof）代替旧的 AOF 文件（这里是temp-rewriteaof-bg-pid.aof）。 如果中途有失败，则直接删除临时文件

### 巡检任务

1.  检查用户是否执行了BGREWRITEAOF命令，有的话server.aof_rewrite_scheduled的flag会打开，执行rewriteAppendOnlyFileBackground函数在后台开始AOF重写。
2.  检查是否AOF是否打开（server,aof_child_pid != -1）且运行结束（wait3检查子进程状态，查看返回的pid是否为aof的child_pid），是的话运行backgroundRewriteDoneHandler函数，它会把新请求的缓存写入到临时文件，然后重命名临时文件为正式的AOF文件，修改aof的文件描述符，根据AOF设置 ，决定是立刻fsync还是后台fsync，最后后台关闭旧文件
3.  及时处理AOF文件过大问题：用两个变量维护了AOF文件当前大小（aof_current_size）和上一次更新的大小（aof_current_size），如果增长超过了我们设定的百分比（默认100%），则启动AOF重写，将最新的命令写进新的AOF文件。

**【注意：Redis 2.6是定时检查是否完成，新版本应该是子进程完成后通过信号通知父进程。而且Redis 2.6中AOF和RDB是不可以同时使用的，bgcommand函数可见如果aof在运行，会直接返回。】**

### 缓存刷新

flushAppendOnlyFile函数，介绍如下：

```c
/* Write the append only file buffer on disk.

 \* 将 AOF 缓存写到文件中
 \* Since we are required to write the AOF before replying to the client,
 \* and the only way the client socket can get a write is entering when the
 \* the event loop, we accumulate all the AOF writes in a memory
 \* buffer and write it on disk using this function just before entering
 \* the event loop again.

 \* 因为程序需要在会回复客户端之前对 AOF 执行写操作。
 \* 而客户端能执行写操作的唯一机会就是在事件 loop 中，
 \* 因为程序将所有 AOF 写保存到缓存中，
 \* 并在进入事件 loop 之前，将缓存写入到文件中。

 \* About the 'force' argument:

 \* 关于 force 参数：
 \* When the fsync policy is set to 'everysec' we may delay the flush if there
 \* is still an fsync() going on in the background thread, since for instance
 \* on Linux write(2) will be blocked by the background fsync anyway.
 \* When this happens we remember that there is some aof buffer to be
 \* flushed ASAP, and will try to do that in the serverCron() function.
 \*
 \* 当 fsync 策略是“每秒进行一次 fsync”时，
 \* 后台队列里可能会有 fsync 等待执行并阻塞，
 \* 这些 fsync 会在 serverCron() 中执行。
 \*
 \* However if force is set to 1 we'll write regardless of the background
 \* fsync. 
 \*
 \* 但是，如果 force 为 1 ，那么不管后台任务是否在 fsync ，
 \* 程序都直接执行 fsync 。
 */
```

Redis会先查看后台线程在排队等待fsync的数量，如果AOF的同步策略是每秒一次且不强制现在同步，那么Redis会尝试推迟同步刷新数据到磁盘（因为如果有fsync在运行，那么我们再调用write系统调用，write就会被后台的fsync阻塞住，从而阻塞了单线程Redis的正常运行），尝试如下：

-   是否排队等待fsync的线程数量不为0？
-   如果是，之前是否推迟过？（aof_flush_postponed_start= 0 ?） 如果没有，记录开始推迟的时间(aof_flush_postponed_start = now())；否则看看上一次推迟是什么时间，如果在两秒内，那我们能接受，继续推迟，否则说明磁盘太忙碌了但我们还是必须得同步了（因为设置的每秒同步刷新一次）。

到这里就是不能推迟必须同步了，aof_flush_postponed_start设为0，然后将AOF缓存写入AOF文件，如果中途失败，

-   一个字节都没写入，那么直接输出错误日志后，退出程序
-   写入了一些，但没写完，那么我们将文件截断（ftruncate函数）至写入缓存之前的大小（回滚），再退出程序。

如果所有字节全部成功写入，那么我们更新AOF文件的当前大小，如果缓存不大，则清空内容用于下次复用，否则释放缓存的所有内存。最后，有需要的话我们执行fsync：

-   如果后台子进程正在重写AOF，那么不执行fsync；
-   否则，如果fsync配置是always，那么必须执行fsync；
-   如果fsync配置是every second并且现在距离上次fsync已经超过一秒了，那么在没有fsync在后台执行时，后台执行fsync。

后台执行任务也是一个有趣的话题，在Redis中后台执行交给bio(background i/o service) 来处理，bio的介绍见bio.c。在Redis 2.6中，后台任务只有两种类型：关闭文件与aof的fsync操作。 Redis用一个结构体表示要执行的工作，而每个类型的工作有一个队列和线程，每个线程都顺序地处理它所对应类型工作队列中的工作。由于多线程操作，因此还会使用锁、条件变量来保证工作执行的唯一性。

创建任务时会将任务加到队列末尾，然后排队计数+1，发送条件变量，表明这个类型来工作了。

处理工作时则是不断读取工作队列的第一个节点（没有节点则通过条件队列阻塞）来处理工作。

工作处理函数是后台任务初始化函数中创建线程时就注册进去的。

涉及主节点同步命令与过期状态到AOF与slaves节点时，会调用feedAppendOnlyFile函数，将命令追加到AOF缓存/文件中。 如果AOF状态为REDIS_AOF_ON，那么说明已经在运行 AOF重写了，调用aofRewriteBufferAppend函数将命令追加到AOF缓存中。



这里的缓存也很值得研究，缓存是一个双链表，每个节点会对应一块连续的内存区域，每次追加命令到缓存，会先找到最后一个节点，尝试追加到对应内存区域，如果不够空间，则将该节点的空间用满，然后创建新节点来追加，直到所有命令都存入缓存为止：

```c
void aofRewriteBufferAppend(unsigned char *s, unsigned long len) {
    listNode *ln = listLast(server.aof_rewrite_buf_blocks);
    aofrwblock *block = ln ? ln->value : NULL;

    while(len) {
        /* If we already got at least an allocated block, try appending
         * at least some piece into it. */
        // 将数据保存到最后一个块里
        // 数据保存的数量是不定的，可能需要创建一个新块来保存数据
        if (block) {
            unsigned long thislen = (block->free < len) ? block->free : len;
            if (thislen) {  /* The current block is not already full. */
                memcpy(block->buf+block->used, s, thislen);
                block->used += thislen;
                block->free -= thislen;
                s += thislen;
                len -= thislen;
            }
        }

        // 分配第一个块，
        // 或者创建另一个块来保存数据
        if (len) { /* First block to allocate, or need another block. */
            int numblocks;

            block = zmalloc(sizeof(*block));
            block->free = AOF_RW_BUF_BLOCK_SIZE;
            block->used = 0;
            listAddNodeTail(server.aof_rewrite_buf_blocks,block);

            /* Log every time we cross more 10 or 100 blocks, respectively
             * as a notice or warning. */
            numblocks = listLength(server.aof_rewrite_buf_blocks);
            if (((numblocks+1) % 10) == 0) {
                int level = ((numblocks+1) % 100) == 0 ? REDIS_WARNING :
                                                         REDIS_NOTICE;
                redisLog(level,"Background AOF buffer size: %lu MB",
                    aofRewriteBufferSize()/(1024*1024));
            }
        }
    }
}
```

代码很优雅，分配新节点与写入数据没有重复的代码。



AOF加载会调用loadAppendOnlyFile函数来重放AOF。具体而言，Redis读取AOF文件，关闭AOF开关（避免同时读写文件），然后构建一个伪终端，不断地一行行读取命令，解析命令给伪终端执行。 由于这可能是一个长IO的任务，期间还是要定期花时间处理请求，Redis会每1000次循环处理一次外部的读写请求。

-   总结：write、fflush、fsync、fdatasync区别
-   注意：redis在linux系统定义的aof_fsync是fdatasync，其他系统才是fsync

## AOF vs RDB

AOF默认关闭，但如果打开，则优先级更高，因为丢失的数据会比RDB更少（持久性保证更好）

RDB默认打开，用于定时备份，避免完全丢失。
