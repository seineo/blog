---
title: Redis数据持久化
date: 2024-07-27 21:25:10
tags:
---

Redis是一款基于内存存储的键值数据库，那么默认情况下，当Redis服务器重启或者宕机后数据就会丢失。在大多业务场景下，我们都不希望数据丢失，因此Redis提供了数据库持久化的特性，允许数据存储到磁盘以应对服务器宕机或重启的情况。

Redis有两种持久化方案：

-   AOF（Append Only File）：将每一条写操作追加到日志文件中。
-   RDB（Redis Database）：定期生成数据的快照并持久化到磁盘中。

接下来会结合Redis 2.6版本源码来学习Redis是如何实现数据持久化的，这两个方案留存至今，只不过实现细节会有所变化。**AOF与RDB的大致流程相似，因此本文主要会分析AOF的实现流程**，并在最后讨论AOF与RDB两种方案的区别以及如何选择。

## 预备知识

Redis关于持久化的实现中涉及了大量数据写入文件的优化，这涉及了write、fwrite、fflush、fsync等函数，这里先简要介绍下，了解了这些才能更好地理解为什么Redis要这么做。

<img src="https://oss.seineo.cn/images/whiteboard_exported_image (1).png" alt="whiteboard_exported_image" style="zoom:50%;" />

如上图所示，在一次写操作中可能涉及多个缓冲区。

-   fwrite是C语言提供的库函数，将数据从程序的缓冲区写到C语言库的缓冲区，最后需要调用fflush函数刷新到内核的缓冲区。
-   write是系统调用，直接从程序缓冲区写入到内核的缓冲区。这样看write似乎性能更好，但是write并没有缓冲区的概念，他是直接将参数指定的长度写入内核，对于未知数据长度时，会涉及多次的系统调用。
-   以上两个函数都不是将数据直接写到磁盘，最后都需要使用fsync系统调用将内核缓冲区的数据刷新到磁盘上。

## AOF实现

### 命令添加到AOF

#### 添加到缓存

由于磁盘I/O缓慢，因此Redis并不是直接将命令写入AOF文件，而是先写入缓存，而后缓存根据配置的策略定期刷入AOF文件中。这一过程由`feedAppendOnlyFile`函数完成：

<img src="https://oss.seineo.cn/images/202408042351057.png" alt="截屏2024-08-04 23.51.55" style="zoom:50%;" />

如上图所示，AOF打开时，执行的更新命令会写入AOF缓存，如果AOF在重写，那么还需要写入AOF的重写缓存。这里需要区分**AOF缓存**与**AOF重写缓存**，AOF缓存每次写操作都会触发，根据配置决定何时同步到磁盘。而AOF重写缓存仅在 AOF 重写进行时写入，防止重写期间的数据丢失，在AOF重写完成后刷新到磁盘。

#### 缓存的实现

这里的缓存也很值得研究，缓存是一个双链表，每个节点会对应一块连续的内存区域，每次追加命令到缓存，会先找到最后一个节点，尝试追加到对应内存区域，如果不够空间，则将该节点的空间用满，然后创建新节点来追加，直到所有命令都存入缓存为止。代码很优雅，这里直接贴上来：

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

#### 刷新到磁盘

缓存写入AOF文件是通过`flushAppendOnlyFile`函数完成的：Redis会先查看后台线程在排队等待fsync的数量，**如果AOF的同步策略是每秒一次且不强制现在同步，那么Redis会尝试推迟同步刷新数据到磁盘，这时因为如果有fsync在运行，则是要将内核缓冲区数据刷新到磁盘，那么我们再调用write系统调用，想写入内核缓冲区，write就会被后台的fsync阻塞住，从而阻塞了单线程Redis的正常运行。**推迟刷新的尝试如下：

1.   是否排队等待fsync的线程数量不为0？

2.   如果是，之前是否推迟过？如果没有，记录开始推迟的时间；否则看看上一次推迟是什么时间，如果在两秒内，那我们能接受，继续推迟，否则说明磁盘太忙碌了但我们还是必须得同步了（因为设置的每秒同步刷新一次）。

能推迟则直接返回，否则必须同步了，通过write系统调用将AOF缓存写入AOF文件，如果中途失败：

-   一个字节都没写入，那么直接输出错误日志后，退出程序。
-   写入了一些，但没写完，那么我们将文件截断（ftruncate函数）至写入缓存之前的大小（回滚），再退出程序。

如果所有字节全部成功写入，那么我们更新AOF文件的当前大小，如果缓存不大，则清空内容用于下次复用，否则释放缓存的所有内存。最后，有需要的话我们执行fsync：

-   如果后台子进程正在重写AOF，那么不执行fsync；
-   否则，如果fsync配置是always，那么必须执行fsync；
-   如果fsync配置是every second并且现在距离上次fsync已经超过一秒了，那么在没有fsync在后台执行时，后台执行fsync。

后台执行任务也是一个有趣的话题，在Redis中后台执行交给bio（background i/o service）来处理。在Redis 2.6中，后台任务只有两种类型：**关闭文件**与**AOF的fsync操作**。 Redis用一个结构体表示要执行的工作，而每个类型的工作有一个队列和线程，每个线程都顺序地处理它所对应类型工作队列中的工作。由于存在多线程操作任务队列的情况，因此会使用锁、条件变量来保证工作执行的唯一性。

```c
struct bio_job {
    // 任务创建时的时间
    time_t time; /* Time at which the job was created. */
    // 任务的参数，如果需要多于三个时，可以传递数组或者结构
    /* Job specific arguments pointers. If we need to pass more than three
     * arguments we can just pass a pointer to a structure or alike. */
    void *arg1, *arg2, *arg3;
};

// 工作线程
static pthread_t bio_threads[REDIS_BIO_NUM_OPS];
static pthread_mutex_t bio_mutex[REDIS_BIO_NUM_OPS];
static pthread_cond_t bio_condvar[REDIS_BIO_NUM_OPS];
// 存放工作的队列
static list *bio_jobs[REDIS_BIO_NUM_OPS];
```

创建任务：将任务加到队列末尾，然后排队的任务数目递增，广播这一条件，表明这个类型来工作了。

处理工作：不断读取工作队列的第一个节点来处理工作，如果没有节点则通过条件变量阻塞，当有节点时被唤醒。

### AOF加载

AOF加载会调用`loadAppendOnlyFile`函数来重放AOF。具体而言，Redis读取AOF文件，关闭AOF开关（避免同时读写文件），然后构建一个伪终端，不断地一行行读取命令，解析命令给伪终端执行。 由于这可能是一个长IO的任务，期间还是要定期花时间处理请求，Redis会每1000次循环处理一次外部的读写请求。

### AOF重写流程

AOF文件文件过大时加载数据会比较缓慢，因此Redis实现了AOF文件的重写，实现在`rewriteAppendOnlyFileBackground`函数：

<img src="https://oss.seineo.cn/images/202408042355277.png" alt="image-20240804235504239" style="zoom: 15%;" />

`rewriteAppendOnlyFileBackground`函数会fork一个子进程，在子进程中调用`rewriteAppendOnlyFile`函数重写命令到临时的AOF文件（temp-rewriteaof-bg-{pid}.aof）中，父进程负责处理错误，如果创建子进程成功则更新服务器关于AOF的状态（如aof_child_pid，不为-1说明后台正在执行aof）。



<img src="https://oss.seineo.cn/images/202408042354269.png" alt="截屏2024-08-04 23.54.07" style="zoom:50%;" />

`rewriteAppendOnlyFile`函数会尝试将每个数据库的键值SET命令保存在自己的临时文件中（temp-rewriteaof-{pid}.aof）。全部成功写完后，fflush刷新用户缓冲区写到内核缓冲区，fsync刷新内核缓冲区确保数据都写入了文件，最后将重写后的 AOF 文件（这里是temp-rewriteaof-{pid}.aof）重命名覆盖旧 AOF 文件（这里是temp-rewriteaof-bg-{pid}.aof）。 如果写入AOF文件过程中有错误，则直接删除临时文件。

### 巡检任务

Redis会有一些巡检任务，根据服务的状态开启AOF的重写或刷新AOF缓存：

1.  检查用户是否执行了BGREWRITEAOF命令，有则执行`rewriteAppendOnlyFileBackground`函数在后台开始AOF重写。
2.  检查是否AOF是否打开且运行结束，是的话运行`backgroundRewriteDoneHandler`函数，它会把新请求的**重写缓存**写入到临时文件（write系统调用），然后重命名临时文件为正式的AOF文件，根据AOF设置决定是立刻fsync还是后台fsync将数据从内核缓冲区真正刷新到这一文件中，最后后台关闭旧文件，避免阻塞服务器。如果刷新缓存或者函数内其他操作出现错误，都会清除缓存、删除临时文件，等待下次重写。
3.  及时处理AOF文件过大问题：用两个变量维护了AOF文件当前大小和上一次更新的大小，如果增长超过了我们设定的百分比（默认100%），则启动AOF重写，将最新的命令写进新的AOF文件。

**注意：Redis 2.6是定时检查是否完成，较新版本应该是子进程完成后通过信号通知父进程。**

## AOF vs RDB

|            | AOF                                                | RDB                                                          |
| ---------- | -------------------------------------------------- | ------------------------------------------------------------ |
| 数据安全性 | 可以配置always或every second，故障时丢失数据较少。 | 完整的数据备份，时间间隔一般会设置得更大，故障时丢失数据较多。 |
| 恢复速度   | AOF文件容易变得比较大，恢复速度较慢。              | RDB是二进制压缩存储，恢复速度较快。                          |
| 适合场景   | 保证数据不丢失。                                   | 数据备份。                                                   |

在实际生产中，根据业务场景综合以上因素选择持久化的选项，也可以两者结合使用。
