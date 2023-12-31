---
title: Prometheus查询
date: 2023-09-27 15:41:28
tags:
  
---

在使用Locust进行压测或使用Kiali监控服务时，都会有着平均延迟、第95百分位(95th percentile，简写为P95)延迟这些概念。平均延迟好理解，P95延迟是什么意思呢？为什么需要这个时间？

<img src="https://oss.seineo.cn/images/202311112153400.png" alt="image-20230921151042328" style="zoom:50%;" />

P95延迟表示95%的请求延迟低于这个值，这个值可以让我们对网络或应用程序的实际性能特征有了更加现实和直观的理解，使我们的应用能够迎合更广泛的受众，而平均响应时间则可能会因为一些极端的值而产生严重偏差。

开源工具可以帮助我们计算这些指标，那么如果我们自己想通过Prometheus收集，又该如何使用PromQL来表示呢？kiali中计算P95响应时间的PromQL为：

![截屏2023-09-27 10.59.53](https://oss.seineo.cn/images/202311112153423.png)

为了明白这一查询语句的意思，需要先学习Prometheus查询的基本概念。

## 数据类型

在Prometheus的查询语言PromQL中，一个表达式可以表达以下四种数据类型的一种：

- 瞬时向量（Instant vector）：共享相同时间戳的时间序列集合，包含满足条件的各时间序列的一个样本。未指定时间戳则返回当前时间的指标值，如`http_requests_total`就返回当前时刻所有的`http_requests_total`指标。
- 区间向量（Range vector）：指定时间窗口的时间序列集合。
- 标量（Scalar）：浮点数值。
- 字符串（String）：字符串值，暂未使用。

## 时序选择器

常用的选择器就两种：

- 瞬时向量选择器：指标名+标签，如`http_requests_total{job="prometheus",group="canary"}`，其中`http_requests_total`是指标名，`{a=1,b=2}`这类大括号中用逗号分隔的列表就是添加的标签。因此这一查询语句的含义就是找到标签job为prometheus、group为canary的`http_requests_total`指标。
- 区间向量选择器：与瞬时向量选择器类似，在语法上，只需在选择器末尾附加一个方括号，以指定向前提取的时间窗口大小。如`http_requests_total{job="prometheus",group="canary"}[5m]`就是获取对应指标前5分钟时间窗口的时序数据。

## 操作符

PromQL支持算术、逻辑、比较等运算符，为了解释前文中的查询语句，这里重点介绍聚集运算符（Aggregation operators），用于按某些维度聚集指标向量并生成新的结果向量。比如

- sum：求和
- min：最小值
- max：最大值
- avg：平均值
- count：计数
- ...

## 指标类型

**注**：指标的类型只是客户端库中的概念，Prometheus服务端存储指标时并不区分。

Prometheus一共有四种指标：

- counter：一个累积的指标，它表示一个单调递增的计数器，其值只能增加或重置为零。例如，可以使用counter表示服务接受的请求数、已完成的任务或错误的数量。

- guage：用于表示可以任意上下的单一指标，如运行的进程数、 内存占用等。
- histogram：采样观察值(通常是请求持续时间或响应大小) ，并在对应范围的桶中计数，得到数据分布。对于一个名为`<basename>`的指标，histogram会在一次抓取的过程中暴露以下三个时序指标：
  1. `<basename>_bucket`：对应桶的计数，**注意histogram为累积直方图**，使用`{le="<upper inclusive bound>"}来得到小于等于某上界的频数，这一指标为counter类指标。
  2. `<basename>_sum`：观测值的总和。
  3. `<basename>_count`：观测到的事件数，等价于`<basename_bucket{le="+Inf"}`。
- summary：与histogram功能类似但是比histogram出现的早，更推荐histogram，因为histogram是可聚集（aggregatable）的而summary不可聚集。另一个不同在于，histogram是在服务端进行计算的，而summary在客户端计算。

## 函数

这里介绍几个常用的、而且前文中查询语句用到了的函数。

### rate与irate

![截屏2023-09-26 20.37.06](https://oss.seineo.cn/images/202311112153483.png)

rate是计算时间窗口内指标每秒的**平均增长率**，比如：

```sql
rate(http_requests_total[5m])
```

是取前5分钟时间窗口数据中的最后一个点和第一个点，相减再除以整个时间长度，那么也就获得每秒新增的http请求数了。

而irate是计算时间窗口最后的**瞬时增长率**，比如

```sql
irate(http_requests_total[5m])
```

是只取了前5分钟时间窗口中的最后两个点，相减再除以两点的时间间隔。

总结：

- rate适合观察长时间的趋势以及用于告警，因为瞬时增长并不一定是异常。
- irate适合观察实时变化，若使用rate计算快速变化的样本平均增长率，容易陷入**长尾问题**（这个问题也可以通过看histogram或者summary来分析解决），因为它用平均值将峰值削平了，无法反映时间窗口内样本数据的快速变化。

### histogram_quantile

`histogram_quantile(φ scalar, b instant-vector)`计算一个直方图瞬时向量b的百分比φ(0<=φ<=1)，比如φ=0.95，就表示计算P95。 以指标`http_request_duration_seconds`为例，它的指标类型为histogram，如前文所数，它会暴露一个名为`http_request_duration_seconds_bucket`的指标，记录了各个桶的频数分布。由于`xx_bucket`为counter类型指标，我们就可以使用以下PromQL得到**前10分钟平均新增请求持续时间的分布**：

```sql
rate(http_request_duration_seconds_bucket[10])
```

如果我们希望获取不同命名空间总的请求持续时间的分布，则可以使用聚集操作：

```sql
sum by (le, namespace) rate(http_request_duration_seconds_bucket[10])
```

可以看到我们除了按namespace聚集，还按le聚集。对于Prometheus的直方图（conventional histogram而非实验性质的native histogram)，聚集操作都必须带上这一个标签，因为histogram是累积直方图，每个桶都有le标签，需要按桶聚集，不然没有意义。

更进一步，如果希望计算不同命名空间中指标`http_request_duration_seconds`的P90值，则可以使用下面的PromQL：

```sql
histogram_quantile(0.9, sum by (le, namespace) rate(http_request_duration_seconds_bucket[10]))
```

那么`histogram_quantile`又是如何实现的呢？

![截屏2023-09-26 20.26.16](https://oss.seineo.cn/images/202311222030173.png)

根据[Prometheus 2.48.0源码](https://github.com/prometheus/prometheus/blob/main/promql/quantile.go#L155)，我们可以发现，虽然histogram在用户看来是累积直方图，但是内部还是用直方图的方式实现（每个存储桶有上下界），且实现百分比预估的方法也十分简单粗暴——假定每个存储桶的数据遵循线性分布。

以上图为例，我们来简单演示一遍Prometheus计算P90的流程。相关函数为：

```go
func histogramQuantile(q float64, h *histogram.FloatHistogram) float64
```

其中q就是上文中的百分比φ(0<=φ<=1)，h为直方图瞬时向量。

1. 由于q>=0.5, 我们反向遍历直方图，计算数据逆向排位：`rank = (1-q) * sum = 0.1 * 30 = 3`。

   ```go
   if math.IsNaN(h.Sum) || q < 0.5 {
     it = h.AllBucketIterator()
     rank = q * h.Count
   } else {
     it = h.AllReverseBucketIterator()
     rank = (1 - q) * h.Count
   }
   ```

2. 寻找数据在倒数第几个存储桶，可以发现就在最后一个。

   ```go
   for it.Next() {
     bucket = it.At()
     count += bucket.Count
     if count >= rank {
       break
     }
   }
   ```

3. 计算数据在该桶的正向排位：`rank = count - rank = 10 - 3 = 7`。

   ```go
   if math.IsNaN(h.Sum) || q < 0.5 {
     rank -= count - bucket.Count
   } else {
     rank = count - rank
   }
   ```

4. 假定该桶数据服从线性分布，根据上下界与该数据的正向排位得到值为`20 + (25 - 20) * (7/10) = 23.5`。

   ```go
   return bucket.Lower + (bucket.Upper-bucket.Lower)*(rank/bucket.Count)
   ```

## 查询语句回顾

有了上述知识，我们就可以弄清文章开头提到的PromQL的含义了。

![截屏2023-09-27 11.00.30](https://oss.seineo.cn/images/202311112153522.png)

上图解构了给出的PromQL，我们一步一步看。

最外围的`histogrm_quantile`是计算直方图的百分位数， 0.95就是要计算第95百分位， 而一长串的`sum by (le) ( rate( istio_request_duration_milliseconds_bucket {reporter=“source”, kubernetes_namespace=“YOUR_NAMESPACE”}[5m] ) ) `就是直方图瞬时向量，其中`istio_request_duration_milliseconds`是histogram类型的向量，暴露出了`istio_request_duration_milliseconds_bucket`这一counter类型的向量，代表直方图各桶的计数，该向量筛选了两个标签：

1. `reporter="source"`表示只要http请求方反馈的该指标，而不要目标方的。
2. `kubernetes_namespace`则指定了命名空间。

`xx_bucket`是counter类型，也就是只增不减， `rate(xx)[5m]`则求出了5分钟时间窗口内平均每秒增长的请求时间，实际上也就是前5分钟平均的请求时间分布， `sum by (le)`则是求和聚集了满足标签要求的指标。

**让我们用一句话描述这一语句的含义：获取前5分钟在命名空间`YOUR_NAMESPACE`中请求方istio代理采集到的P95请求时延**。

至此，我们了解了如何使用PromQL在Prometheus中查询百分位的请求延迟，了解了这类复杂的查询语句，对于其他查询语句也能很快上手了。
