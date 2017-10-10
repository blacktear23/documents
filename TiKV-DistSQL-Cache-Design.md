# TiKV DistSQL Cache

## 起源

MySQL 有个 Query Cache，那么 TiDB 呢？

## DistSQL

当我们在 TiDB 执行一个查询语句的时候，TiDB 会根据查询语句计算出需要扫描的 KeyRange 然后根据 KeyRange 分布的 Region 生成 N 个 DistSQL 发送到 TiKV 中执行。

由此，DistSQL 里面包含了查询条件和要扫描的 Region 的 KeyRange。那么如果需要为 DistSQL 增加 Cache 的话，区分不同的 DistSQL 只需要把查询条件和要扫描的 KeyRange 编码成一个 Key 就可以了。

目前的代码实现为把 KeyRange 和 DistSQL 的 Executors 编码成字符串。

## Cache

LRU Cache 目前来看是个比较简单高效的 Cache 策略，但是 DistSQL 使用的 LRU Cache 并不是个简单的 LRU Cache 就可以的，需要对 LRU 算法进行小调整：

* Cache Item 的数量需要 LRU 算法进行自动淘汰
* Cache Item 需要记录 DistSQL 对应的 RegionID
* 需要提供针对 RegionID 的 Cache 失效功能

在实现了上述功能之后，剩下的就是在 Region 数据变动的地方加入 Hook 函数，调用针对 RegionID 失效相关 Cache Item 的函数。需要 Region 级别失效的时刻为：

* RocksDB 写入时
* Region Split 时
* Raft Log Apply 时 （目前还不太确认，按说 Raft Log Apply 时应该会有对应的 RocksDB 写入，不过为了安全起见，这个时间点还是加入了针对 RegionID 失效的 Hook）

另外 RegionInfo 也会保存到 Cache Item 当中。一旦 Region 的 epoch 和 conf_epoch 改变了，那么当前 Cache Item 应该被认为未命中。

## Cache 什么数据

对于 TiKV 来说，RocksDB 自身会有 Cache，那么 DistSQL Cache 应该只缓存 RocksDB 的 Cache 无法缓存的内容。根据 coprocessor 所提供的功能，选择 DistSQL 中包含 TopN 和 Aggregation 的 DistSQL 结果来缓存是比较合适的。因为这种 DistSQL 会对 RocksDB 取出来的数据进行大量计算，同时计算出来的结果也会比较小。

对于计算结果非常大的 DistSQL 来说，网络传输所占用的时间大概率会成为性能瓶颈（近乎全量导出数据到 TiDB 端），所以目前非 TopN，Aggregation 的 DistSQL 不予缓存。

## Cache 的特性

在实践过程中发现一个 DistSQL Cache 的特性。由于 DistSQL 是根据 Region 和 KeyRange 进行区分的，那么当 OLAP 的查询语句中 Where 子句中的条件不变，只是扫描的行数增加或者减少，那么之前的 Cache 也会命中。例如：

如果表结构为：

```
CREATE TABLE test (
    `id` INT PRIMARY KEY,
    `domain_name` VARCHAR(255),
    `date` VARCHAR(10),
    `count` BIGINT,
);
```
第一次查询语句为：`SELECT SUM(count) FROM test WHERE id BETWEEN 0 AND 1000 GROUP BY date`

第二次查询语句为：`SELECT SUM(count) FROM test WHERE id BETWEEN 0 AND 2000 GROUP BY date`

同时 0\~1000 在 Region 1，1001\~2000 在 Region 2。那么第二次查询会命中 Region 1 的 Cache，同时计算 Region 2 的结果。因此，计算时间大约会减半。

## Cache 未来的改进

以下是针对 LRU Cache 的一些改进想法，还未实现：

* 针对 Cache 所占用的内存大小进行 Cache Item 的淘汰，以达到固定内存大小的限制
* 使用 TwoQueue Cache 提高缓存命中率
* 针对锁的优化，把锁的粒度细化到 Region 级别，从而提高并发能力
* 针对 DistSQL 结果的大小选择性缓存

## 监控项

为了能看到 Cache 占用的内存和 Item 数量，需要增加 metrics。Cache Item 占用的内存是在 Cache Item 中记录了 Key 和 Value 外加额外数据结构的字节大小实现的。目前的实现可以获取一个基本正确的缓存字节大小，但并不能非常精确（比如精确到 1 Byte）

## 线上测试结果

DistSQL Cache 可以提高重复查询的响应速度，同时对于聚合和 TopN 类型的查询其结果占用的字节数也不会很大，1000 个 Cache Item 所占用的内存基本上在 50MB 以下。

由于线上项目的 Web 服务和数据导入服务是两套程序，所以在 Web 服务中使用 Memcached 进行缓存会引入复杂的缓存失效策略。同时对比 DistSQL Cache 中的失效粒度来看，Memcached 的缓存失效粒度更大，二期对于重复计算没有更好的优化。

目前线上项目的用法有两种：

1. 查询一天的各个域名的访问情况，并根据某个指标排序，同时提供翻页。其 SQL 类似：`SELECT * FROM some_table WHERE id BETWEEN 201709010000000000 AND 20170901999999999 ORDER BY some_column DESC LIMIT 0, 20`
2. 按天聚合所有域名的访问情况。其 SQL 类似：`SELECT SUM(column_a), SUM(column_b) ... FROM some_table WHERE id BETWEEN 201709010000000000 AND 201709079999999999 GROUP BY date`

在第一种场景下，未命中缓存的情况下，API 响应速度在 1s ~ 2s，命中缓存的情况下，API 响应速度在 100ms ~ 400ms 。

在第二种场景下，未命中缓存的情况下，API 响应速度受到日期范围的影响，一个月数据的聚合在冷数据和未命中缓存情况下 API 响应时间会大于 30s，如果一个月的缓存已建立，那么选择半个月时间段聚合的话，API 响应时间会在 ms 级别。如果新的聚合时间区间比已建立的缓存时间段多 1 天的话，API 响应时间会在 1s ~ 3s。