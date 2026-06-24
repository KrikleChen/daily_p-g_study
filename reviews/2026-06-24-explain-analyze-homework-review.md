# 2026-06-24 EXPLAIN ANALYZE 练习批改记录

## 批改对象

- 练习文件：`exercises/explain-analyze-order-revenue.md`
- 主题：用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 分析订单收入统计 SQL
- 验证数据库：本地 PostgreSQL 18.4，数据库 `test`

## 本地验证方式

为了避免影响 `public` schema 里的作业表，先创建临时 schema `codex_explain_check`，在里面重新建表、插入数据、执行加索引前后的执行计划。验证完成后已经删除临时 schema。

另外也只读检查了 `public` schema 当前状态：

- `users`：10000 行
- `products`：1000 行
- `orders`：100000 行
- `order_items`：300000 行

`public` schema 当前已经存在这些索引：

- `idx_orders_status_created_user`
- `idx_order_items_order_id`
- `idx_order_items_product_id`
- `idx_products_category`

所以 `public` schema 当前只能直接复核“加索引后”的执行计划。

## 本地复核到的加索引后计划摘要

在 `public` schema 上重新执行同一条 SQL，观察到：

```text
Execution Time: 612.574 ms
Planning Time: 6.854 ms
orders: Bitmap Heap Scan + Bitmap Index Scan
order_items: Seq Scan
products: Seq Scan
users: Seq Scan
JOIN: 3 个 Hash Join
Aggregate: GroupAggregate
Sort Method: quicksort
Buffers: shared hit=3450
```

## 你已经掌握得不错的地方

你已经能识别这些关键执行计划节点：

- `Seq Scan`
- `Bitmap Heap Scan`
- `Bitmap Index Scan`
- `Hash Join`
- `GroupAggregate`
- `Sort`
- `Limit`

你也已经知道要做加索引前后的对比，这个方向是对的。

当前水平判断：已经入门，能看到主要节点，但还需要练习把每个数字准确归属到对应节点。

## 需要修正的地方

### 1. `users` 的估算行数写错了

你写的是：

- `users` 估算行数：100000
- `users` 实际行数：10000

这里应该修正为：

- `users` 估算行数：10000
- `users` 实际行数：10000

读计划时要注意这种格式：`Seq Scan ... rows=10000 ... actual ... rows=10000`。

前面的 `rows=10000` 是优化器估算行数，后面的 `actual ... rows=10000` 是实际执行行数。

### 2. 过滤条件归属写串了

`Rows Removed by Filter: 81772` 属于 `orders` 表过滤，不属于 `products`。

它表示 `orders` 表里被下面两个条件一起过滤掉的行数：

- `o.status = 'paid'`
- `o.created_at >= now() - interval '90 days'`

`Rows Removed by Filter: 500` 属于 `products` 表过滤。

它表示 `products` 表 1000 行里，有 500 行不是 `book` 或 `course`，所以被过滤掉，留下 500 行。

### 3. 聚合后的实际行数不是 2

计划里常见这种形式：`GroupAggregate ... rows=2 ... actual ... rows=5`。

这里：

- `rows=2` 是优化器估算的聚合后行数。
- `actual ... rows=5` 是真实聚合后输出行数。

这个练习里数据有 5 个城市，所以实际输出 5 个城市分组是合理的。

### 4. `HAVING` 过滤的是分组，不是明细行

`HAVING sum(...) > 50000` 发生在 `GROUP BY` 之后。

它过滤的不是 `order_items` 明细行，而是城市分组。例如某个城市的总收入小于 50000，这个城市整组会被过滤掉。

如果 `GroupAggregate` 节点下面没有出现 `Rows Removed by Filter`，通常说明没有分组被 `HAVING` 过滤掉。

## 你的问题解答

### 为什么 3 个 JOIN 都是 Hash Join？

不是只因为它们是等值连接，但等值连接是 `Hash Join` 可以使用的重要前提。

更准确地说：

- 这 3 个 JOIN 都是等值连接，例如 `oi.order_id = o.id`。
- 数据量比较大，`Nested Loop` 可能需要大量重复查找。
- 输入没有合适的排序，`Merge Join` 未必划算。
- 优化器估算后认为 `Hash Join` 成本更低。

所以这里选择 `Hash Join` 是合理的。

### top-N heapsort 是什么？

`top-N heapsort` 是 PostgreSQL 在 `ORDER BY ... LIMIT N` 场景下可能使用的排序方式。

它不一定完整排序所有结果，而是维护当前最靠前的 N 条记录。

这次计划里没有出现 `top-N heapsort`，是正常的。因为聚合后只有 5 行，而 `LIMIT 10`，输出行数比限制还少，普通 `quicksort` 就够了。

### shared hit 是什么？

`shared hit` 表示 PostgreSQL 从 shared buffers 里命中了数据页。

可以理解成 PostgreSQL 自己的缓存命中。命中 shared buffers 时，不需要再把这个数据页读入 PostgreSQL 缓存。

### shared read 是什么？

`shared read` 表示 PostgreSQL 需要把数据页读入 shared buffers。

它不一定等于真实物理磁盘 IO，因为可能来自操作系统缓存。但从 PostgreSQL 角度看，这是一次 read。

默认一个 PostgreSQL page 通常是 8KB。比如 `shared hit=3450`，大概表示访问了 3450 个 8KB 页面。

## 加索引后的正确理解

加索引后，`orders` 从顺序扫描变成下面这种结构：

```text
Bitmap Heap Scan on orders
  -> Bitmap Index Scan on idx_orders_status_created_user
```

这说明 `idx_orders_status_created_user` 被优化器使用了。

但是：

- `order_items` 仍然是 `Seq Scan`
- `products` 仍然是 `Seq Scan`
- `users` 仍然是 `Seq Scan`
- 3 个 JOIN 仍然是 `Hash Join`
- 聚合仍然是 `GroupAggregate`

所以不能简单写“加索引以后一定变快了”。

更准确的结论应该是：

> 加索引后，`orders` 表的过滤条件可以使用组合索引，扫描方式从顺序扫描变成了 `Bitmap Heap Scan + Bitmap Index Scan`。但是查询仍然需要扫描 30 万行 `order_items` 并做多次 `Hash Join` 和聚合，所以整体耗时不一定大幅下降。执行时间还会受到缓存、随机数据分布和重复运行次数影响。

## 建议你改写的对比表

| 对比项 | 加索引前 | 加索引后 | 我的理解 |
| --- | --- | --- | --- |
| `Execution Time` | 以自己第一次计划为准 | 当前复核约 612 ms | 不能只看一次执行时间，要结合缓存和计划节点 |
| `Planning Time` | 以自己第一次计划为准 | 当前复核约 6.854 ms | 建索引后优化器可选路径更多，规划时间可能增加 |
| `orders` 扫描方式 | `Seq Scan` | `Bitmap Heap Scan` | 组合索引用上了 |
| `order_items` 扫描方式 | `Seq Scan` | `Seq Scan` | 需要参与大量 JOIN，全表扫仍然合理 |
| `products` 扫描方式 | `Seq Scan` | `Seq Scan` | 表很小，且命中 50%，全表扫合理 |
| `users` 扫描方式 | `Seq Scan` | `Seq Scan` | 1 万行小表，hash 后参与 JOIN |
| 主要 JOIN 类型 | `Hash Join` | `Hash Join` | 等值连接且数据量较大，hash join 合理 |
| 聚合方式 | `GroupAggregate` | `GroupAggregate` | 需要按 `city, id` 排序后聚合 |
| `shared hit` | 以自己第一次计划为准 | 当前复核 `3450` | 命中 shared buffers 的数据页 |
| `shared read` | 以自己第一次计划为准 | 当前复核 `0` | 当前这次数据都已在缓存里 |

## 下一步训练方法

以后读执行计划，先按这个顺序标注每个节点：`节点类型 -> 属于哪张表/哪一步 -> 估算 rows -> 实际 rows -> Buffers -> 我的解释`。

不要先写结论，先把每个节点归属清楚。

## 对你目前掌握程度的评价

当前掌握程度：`60/100`。

具体判断：

- 已经能识别主要执行计划节点。
- 已经知道要比较加索引前后。
- 对 `Hash Join`、`Seq Scan`、`GroupAggregate` 有初步认识。
- 还需要强化估算值和实际值的区分。
- 还需要练习不要把一个节点的 `Rows Removed by Filter` 写到另一个节点下面。
- 对 `Buffers` 还处于刚开始理解阶段。

下一阶段目标：

- 能准确标注每个节点属于哪张表或哪一步。
- 能说清楚每个节点的估算行数和实际行数。
- 能判断某个索引是否真的被使用。
- 能解释为什么某些索引建了但没有被使用。
