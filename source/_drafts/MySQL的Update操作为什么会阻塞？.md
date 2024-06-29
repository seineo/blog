---
title: MySQL的Update操作为什么会阻塞？
tags:
---

### 现象

做实验，发现阻塞



### 原因

next-key lock



什么是select for update？ 

因为select for update 到 commit之间可以加业务逻辑。

这是悲观锁处理的常见方案。可以防止第三方在commit之前修改数据。 否则比如select获取数据，然后进行计算操作，最后update，却发现被修改了导致RowsAffected为0。

> **乐观锁、悲观锁**
>
> 乐观锁：乐观地认为大部分情况下其他事务不会并发写。实际上是没有锁的，适合读多写少，自己根据版本号是否是当前版本。 （比如etcd就是基于乐观锁的，因为Kubernetes中etcd主要用于存储元数据，**而元数据的写操作其实是比较少的，但是会有很多的客户端同时 watch 这些元数据的变更。也就是说 etcd 的使用场景是一种 "读多写少" 的场景，etcd 里的一个 key ，其实并不会发生频繁的变更，但是一旦发生变更，etcd 就需要通知监控这个 key 的所有客户端**。
>
> 悲观锁：悲观地认为其他事务会来并发写。



为什么上述语句就要使用next-key lock？



### 总结





### 参考

- https://dev.mysql.com/doc/refman/8.0/en/innodb-next-key-locking.html
- https://youguanxinqing.xyz/archives/149/
