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
