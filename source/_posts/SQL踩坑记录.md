---
title: SQL踩坑记录
tags: 
date: 2024-03-15 16:05:30
---

后端开发时常常会与数据库打交道，无论是写业务逻辑还是配置监控，SQL是必不可少的。

## MySQL语法

### 三值逻辑

关系数据库使用`NULL`来表示**无数据**，即可以有值，但还不知道填什么值。`NULL`可以是任何值，也就无法判断一个值与 `NULL` 的比较结果是真还是假，这就是除了`True`与`False`，第三个逻辑值`Unknown`出现的原因。

`NULL`表示未知的值，那么我们判断一个数据是否是`NULL`，就不能用`a = NULL`这样的写法，因为`NULL`未知，这样写的结果只会是`Unknown`，因此数据库引入IS NULL或IS NOT NULL来判断`NULL`。

习惯二值逻辑的我们，很容易在写查询语句的时候没有考虑到`NULL`，从而有可能导致查询结果不符合预期。比如我们有一张users表，我们希望查出field1字段（允许为`NULL`）不为test的数据，我们很容易写出：

```sql
SELECT *
FROM users
WHERE field1 <> 'test';
```

但这样会遗漏了field1字段为`NULL`的数据。这是因为WHERE查询语句要求逻辑运算结果为`True`才会加入结果集，当遇到field1值为`NULL`时，`NULL <> 'test'`会返回`Unknown`，不符合条件。

因此我们需要加上`NULL`的判断：

```sql
SELECT *
FROM users
WHERE field1 <> 'test' OR field1 IS NULL;
```

遇到`NULL`数据时，运算表达式为`NULL <> 'test' OR NULL IS NULL`， 由于后者为`True`，因此会将该记录放入结果集合。

对于`Unknown`在逻辑运算式的计算规则，只需要记住：**如果结果依赖于Unknown，则结果为Unknown，否则Unknown无影响。**什么叫**“依赖”**呢？比如上面的例子：`Unknown OR True`，或运算符的右边已经为`True`了，根据或运算的计算规则这个运算结果已经确定为`True`了，左边具体是什么已经不重要了，这就是结果不依赖于`Unknown`本身，不影响结果为`True`；如果运算式为`Unknown OR Unknown`，那么运算结果依赖于`Unknown`，返回`Unknown`。

### 分页数据重复

之前在查看分页数据时，发现有时同一条数据会在不同页重复出现，但排查发现页码和页数据量都正确传入，SQL语句也符合预期，SQL可以极致简化为：

```sql
SELECT a, b, c
FROM table_test
LIMIT ps OFFSET p
```

为什么数据会重复出现呢？[官方Q&A](https://bugs.mysql.com/bug.php?id=69732)说到：

>   Without a distinct ORDER BY the result order is undefined.
>   Result order can change as statistics get updated, as index trees are reshuffled, as data is partitioned across different physical machines so that result order depends on network latencies…

也就是说MySQL查询结果顺序会收到数据存储的位置、数据结构等的影响，在没有对唯一字段（比如unique key、primary key）ORDER BY的情况下，结果顺序是不保证的。上述例子就是因为SQL语句并没有对字段排序，导致结果随机返回，只需要加上对**唯一字段**的排序即可。

### 锁定读

遇到一个其实在业务中挺常见的数据库操作需求，可以分为三步：

1.   根据条件查询数据；
2.   对查询出来的数据进行数值判断；
3.   如果符合要求则更改写入数据库。

那我们可能会想，将这三步放入一个事务中执行就确保了操作的安全性。其实不然，我们可以看一个伪代码展示的例子：

```sql
BEGIN;
SELECT balance FROM accounts WHERE user='xiaoming';
if balance > = 500:
	UPDATE accounts SET balance = balance - 500 WHERE user='xiaoming';
	UPDATE accounts SET balance = balance + 500 WHERE user='xiaohong';
COMMIT;
```

逻辑十分简单，我们查询`xiaoming`同学的余额，如果余额不低于500，那么就向`xiaohong`同学转账500元。我们可以想象在另外一个事务中，`xiaoming`也尝试向`xiaoli`同学转账500元，那么当并发执行时，有可能他们都通过了第三行的判断语句，然后排队更新，那么结果就是即使`xiaoming`同学余额只有500块钱，那么在没有数据库完整性（如定义balance字段不小于0）的前提下，`xiaoming`同学的余额会变为-500元，而`xiaohong`和`xiaoli`同学的余额都成功增加了500元。上述结局是银行不想看见的:)

那么为了保证事务中查询操作后的插入与更新操作的安全性，MySQL的InnoDB引擎提供了锁定读（Locking Reads），这里仅是简要介绍，更多细节见[官方文档](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking-reads.html)。

简单来说，锁定读分为两种类型：

1.   `SELECT ... FOR SHARE`：给查询的行设置共享锁，在事务提交前不可被更改。可以用于保证事务中查询操作后的插入操作。
2.   `SELECT ... FOR UPDATE`：给查询的行设置排他锁（相当于UPDATE操作），在事务提交前不可被更改、加读锁等，但是普通的SELECT语句可以读取到这些行，因为SELECT是一致性读（多版本控制，读取的是快照）。可以用于保证事务中查询操作后的更新操作。

**使用时的注意事项：需要在事务中使用，锁在事务提交或回滚后才释放。**

如果某行被事务锁定，则请求同一锁定行的 `SELECT ... FOR UPDATE` 或 `SELECT ... FOR SHARE` 事务必须等待，直到阻塞事务释放行锁。此行为可防止事务更新或删除其他事务查询更新的行。但是，如果我们在有些业务场景希望查询在请求的行被锁定时立即返回（比如抢锁，不是第一个拿到，后面的就可以放弃了），或者如果可以接受从结果集中排除锁定的行，则分别可以加上`NOWAIT`和`SKIP LOCKED`选项。

注意`SELECT ... FOR UPDATE NOWAIT`在请求的行被锁定时，直接返回的是错误：

```SQL
mysql> SELECT * FROM tests where unique_id='unique' FOR UPDATE NOWAIT;
ERROR 3572 (HY000): Statement aborted because lock(s) could not be acquired immediately and NOWAIT is set.
```



## 慢查询

### 索引失效

之前需要用到一个数据库插件，插件对a类型有一些函数支持，但我们原始数据field1的类型为b，因此查询语句为：

```sql
SELECT *
FROM table_test
WHERE a(field1) = 'test'
```

这样执行很缓慢，通过查看执行计划发现，并没有走我们在field1字段设置的索引，这就是索引失效。因此我们需要进行数据迁移，让数据转为a类型存储，这样就可以使用索引来查询了。

让我们总结一下索引失效的常见场景：

1.   未遵循最左前缀匹配，如`field1 LIKE '%abc'`。
2.   使用函数，如`left(field1, 4)`。
3.   表达式计算，如`field1 + 1`。
4.   ……

## Grafana相关

### 常用面板及对应使用场景

| 面板类型    | 使用场景                                                     |
| ----------- | ------------------------------------------------------------ |
| Time Series | 历史变化趋势                                                 |
| Pie         | 各类数据占比                                                 |
| Stat        | 实时数值统计                                                 |
| Bar         | 一般用stacking的bar，这样既可以看到各类数据的具体数值，又可以看到某一类的总数 |

### 分组查询

Grafana查询结果有以下两种类型：

- Time Series：时序数据，一般用于历史趋势分析。最后查询出来的字段需要包含time字段（别名即可）用于横轴，其余的字符串字段会作为label，数值字段作为纵轴数据。
- Table：表格数据，一般用于状态统计。这里有个坑点：**与time series不同，table格式用字符字段group by并将其select出来，它并不会作为一个分组的label，而是不显示，达不到通过该字段筛选分类数据的效果。**以gpu_type为例：

    ```sql
    SELECT gpu_type, count(val)
    FROM table_test
    GROUP BY gpu_type
    ```

    这样写并不会显示各种gpu_type的label。如果想按gpu_type分类，需要显式地用IF ELSE或者CASE WHEN将gpu_type的各种条件分支写出来。

## 参考

- https://dba.stackexchange.com/questions/6051/what-is-the-default-order-of-records-for-a-select-statement-in-mysql
- https://modern-sql.com/concept/three-valued-logic
- https://bugs.mysql.com/bug.php?id=69732
