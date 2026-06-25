# 2026-06-24 EXPLAIN ANALYZE 诊断拷问记录

## 记录目的

这份记录用于追踪 `EXPLAIN ANALYZE` 练习中的薄弱点、追问问题、学员回答和后续修正。

本轮诊断对象：

- `exercises/explain-analyze-order-revenue.md`
- 已有批改记录：`reviews/2026-06-24-explain-analyze-homework-review.md`

当前状态：已出题，等待学员回答。

## 当前暴露的薄弱点

### 1. 执行计划数字归属不够稳定

表现：

- 把 `users` 的估算行数写成 `100000`，但 `users` 实际只有 `10000` 行。
- 能识别节点类型，但还没有稳定区分每个节点自己的 `rows`、`actual rows` 和 `Rows Removed by Filter`。

要训练的能力：

- 看到一个 plan node 时，先判断它属于哪张表或哪一步。
- 再读该节点自己的估算值和实际值。
- 不把父节点、兄弟节点或其他表的指标混到一起。

### 2. 过滤条件和表的对应关系容易混

表现：

- `Rows Removed by Filter: 81772` 属于 `orders` 的过滤。
- `Rows Removed by Filter: 500` 属于 `products` 的过滤。
- 当前作业里有把过滤条件和过滤行数串到不同表下面的迹象。

要训练的能力：

- `Filter` 写在哪个扫描节点下面，就归哪个节点。
- `orders` 的两个过滤条件是 `status` 和 `created_at`。
- `products` 的过滤条件是 `category IN ('book', 'course')`。

### 3. 对估算行数和实际行数的含义还不够牢

表现：

- 能说 `ANALYZE` 是收集统计信息。
- 但在读计划时还容易把 `rows=...` 和 `actual ... rows=...` 混在一起。

要训练的能力：

- `cost=... rows=...` 里的 `rows` 是优化器执行前估算。
- `actual time=... rows=... loops=...` 里的 `rows` 是实际执行结果。
- 判断估算是否准确，要比较估算行数和实际行数，而不是只看节点名字。

### 4. `HAVING` 和聚合后的分组过滤还不清楚

表现：

- 作业中写到“过滤了 3346”，但这更像是把聚合前输入行数、聚合后行数或其他节点输出混在了一起。
- 对 `HAVING` 过滤的是明细行还是分组还不稳定。

要训练的能力：

- `WHERE` 过滤 join/聚合之前的行。
- `GROUP BY` 先把行聚成城市分组。
- `HAVING` 过滤的是聚合后的组，不是 `order_items` 明细行。

### 5. 索引对比结论偏粗

表现：

- 当前能看到 `orders` 从 `Seq Scan` 变成 `Bitmap Heap Scan`。
- 但对“加索引以后为什么整体不一定明显变快”解释还不够完整。
- 对“哪个索引真正起作用、哪个索引没明显作用”还需要更精确。

要训练的能力：

- 对比执行计划时，不能只看是否出现索引。
- 要看扫描节点、JOIN 节点、聚合节点、排序节点、总耗时、Buffers 是否整体变化。
- 小表、低选择度条件、大量 join 输入，都可能让索引收益不明显。

### 6. JOIN 顺序和 JOIN 类型理解还停在表面

表现：

- 能看到 3 个 `Hash Join`。
- 对“是不是因为等值连接所以都是 Hash Join”的理解还需要补一层：等值连接只是前提之一，最终还要看代价、数据量、排序、索引路径和 join 输入大小。

要训练的能力：

- `Hash Join` 适合较大输入的等值连接。
- `Nested Loop` 常见于外表较小、内表有高效索引查找的情况。
- `Merge Join` 通常需要两边按 join key 有序，或者排序成本划算。
- JOIN 顺序会改变中间结果大小，从而影响后续扫描、join、聚合和排序成本。

## 第一轮拷问问题

请学员逐题回答。回答后，本文件需要继续追加“学员回答”和“批改判断”。

### Q1. 扫描节点归属

看到下面这种执行计划片段时：

```text
Seq Scan on public.users u  (cost=0.00..164.00 rows=10000 width=16) (actual time=0.010..1.200 rows=10000 loops=1)
  Output: u.id, u.name, u.city
```

请回答：

- 这个节点属于哪张表？

答案：属于 user 表

- 估算行数是多少？

答案：估算行数为 10000

- 实际行数是多少？

实际行数为 10000 行

- 如果你在别处看到 `orders` 是 100000 行，能不能把 100000 写到这个节点上？为什么？

你这个问题是什么意思？我没理解

学员回答：待填写。

批改判断：待填写。

### Q2. 过滤条件归属

如果计划里出现：

```text
Seq Scan on public.orders o
  Filter: ((o.status = 'paid'::text) AND (o.created_at >= (now() - '90 days'::interval)))
  Rows Removed by Filter: 81772
```

请回答：

- `81772` 是哪张表被过滤掉的行数？

是 orders 表被过滤的行数

- 这个数字是被哪个条件过滤出来的？

被两个条件一起过滤出来的，o.status = 'paid'::text) AND (o.created_at >= (now() - '90 days'::interval)

- 它能不能拿来解释 `products.category` 的过滤？为什么？

不能，这个只是过滤了订单状态和创建时间

学员回答：待填写。

批改判断：待填写。

### Q3. `products` 小表为什么可能 `Seq Scan`

`products` 只有 `1000` 行，条件是 `p.category IN ('book', 'course')`，实际留下约 `500` 行。

请回答：

- 这个条件选择度大概是多少？

50%

- 为什么即使有 `idx_products_category`，优化器也可能继续用 `Seq Scan`？

因为表的数据量太小了

- 这种情况下“没用索引”一定是坏事吗？

不是，用了索引反而会劣化执行效率，因为启动索引也耗时间，需要资源

学员回答：待填写。

批改判断：待填写。

### Q4. 估算行数和实际行数

执行计划中如果看到：

```text
rows=18748
actual ... rows=18228
```

请回答：

- 哪个是估算行数？

18748 是估算行数

- 哪个是实际行数？

18228 是实际行数

- 这两个数差距大不大？

差距不大

- 这种差距对优化器选择执行计划有什么影响？

影响几乎没有，因为偏差很小，如果差距在数量级，就会影响执行计划的生成

学员回答：待填写。

批改判断：待填写。

### Q5. `HAVING` 到底过滤什么

原 SQL 里有：

```sql
GROUP BY u.city
HAVING sum(oi.quantity * oi.unit_price) > 50000
```

请回答：

- `WHERE` 先执行还是 `HAVING` 先执行？

having 先执行吗？为什么？

- `HAVING` 过滤的是 `order_items` 明细行，还是城市分组？

过滤的是明细行，解释一下为什么

- 如果最终有 5 个城市分组，`HAVING` 最多能过滤掉多少个对象？

啥意思？最多过滤 5 个

学员回答：待填写。

批改判断：待填写。

### Q6. 为什么是 `Hash Join`

这条 SQL 里有 3 个等值连接：

```sql
o.user_id = u.id
oi.order_id = o.id
p.id = oi.product_id
```

请回答：

- 等值连接是不是 `Hash Join` 的必要条件之一？

是必要条件之一，给我仔细讲一下 hashjoin

- 为什么不能只说“因为是等值连接，所以一定是 Hash Join”？

因为等值连接还有可能是其他的 join 方式，什么 merge join 啊之类的

- 什么情况下优化器可能改用 `Nested Loop`？

小表驱动大表

学员回答：待填写。

批改判断：待填写。

### Q7. JOIN 顺序为什么重要

请用这道题里的表解释：

- 如果先过滤 `orders` 和 `products`，再 join `order_items`，中间结果可能变大还是变小？

没理解，你能给我讲清楚全流程吗

- 如果先把 `order_items` 这张 30 万行表和其他表 join，为什么成本可能很高？
- JOIN 顺序影响的是最终结果，还是影响执行成本？最终结果会不会变？

学员回答：待填写。

批改判断：待填写。

### Q8. 加索引后怎么判断哪个索引起作用

加索引后看到：

```text
Bitmap Heap Scan on public.orders o
  -> Bitmap Index Scan on idx_orders_status_created_user
```

同时 `order_items`、`products`、`users` 仍然是 `Seq Scan`。

请回答：

- 哪个索引被明确使用了？

idx_orders_status_created_user

- `idx_order_items_order_id` 有没有从这个现象里看出明显使用？

有

- 为什么不能只写“加了索引，查询变快了”？

不明白什么意思

学员回答：待填写。

批改判断：待填写。

### Q9. `Bitmap Index Scan` 和 `Bitmap Heap Scan`

请回答：

- `Bitmap Index Scan` 主要做什么？

扫索引对比

- `Bitmap Heap Scan` 主要做什么？

对比桶内的内容

- 为什么 PostgreSQL 不一定直接用普通 `Index Scan`？

学员回答：待填写。

批改判断：待填写。

### Q10. `Buffers` 的含义

如果看到 `Buffers: shared hit=3450 read=0`。

请回答：

- `shared hit` 表示什么？
- `shared read` 表示什么？
- `read=0` 能不能直接说明这条 SQL 没有任何 IO 成本？为什么？

学员回答：待填写。

批改判断：待填写。

### Q11. `ORDER BY ... LIMIT` 和 `top-N heapsort`

这条 SQL 有：

```sql
ORDER BY revenue DESC
LIMIT 10
```

但如果聚合后只有 5 行。

请回答：

- 为什么可能看不到 `top-N heapsort`？
- `top-N heapsort` 主要优化什么场景？
- 聚合后只有 5 行时，排序是不是这条 SQL 的主要成本？

学员回答：待填写。

批改判断：待填写。

### Q12. 如何做一次合格的索引前后对比

请你设计一个对比表，至少包含：

- 扫描方式变化。
- JOIN 类型变化。
- 聚合方式变化。
- `Execution Time`。
- `Planning Time`。
- `Buffers`。
- 哪个索引被使用。
- 哪些索引没有明显使用。

请回答：你会如何下结论，避免只写“用了索引所以更快”？

学员回答：待填写。

批改判断：待填写。

## 本轮预期评分口径

回答后按以下标准判断：

- `80-100`：能准确把数字归属到节点，能解释优化器选择，不靠背答案。
- `60-79`：能识别主要节点，但解释中仍有少量归属错误或结论过粗。
- `40-59`：能说出一些概念，但估算/实际、WHERE/HAVING、索引作用经常混。
- `0-39`：主要靠猜节点名，不能从执行计划中的具体数字推出结论。

## 下一步记录方式

学员回答后，在每道题下补充：

- 学员回答。
- 正确点。
- 错误点。
- 修正解释。
- 是否需要复训。
