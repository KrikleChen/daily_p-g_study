# 2026-06-25 EXPLAIN ANALYZE 诊断题批改记录

## 批改对象

- 诊断题记录：`reviews/2026-06-24-explain-analyze-diagnostic-drill.md`
- 原练习文件：`exercises/explain-analyze-order-revenue.md`
- 主题：`EXPLAIN ANALYZE` 执行计划数字归属、过滤条件归属、`HAVING`、JOIN、索引对比

## 验证环境

- PostgreSQL：18.4 Homebrew
- 客户端：`/usr/local/opt/postgresql@18/bin/psql`
- 数据库：`test`
- schema：`public`
- 数据量：
  - `users`：10000 行
  - `products`：1000 行
  - `orders`：100000 行
  - `order_items`：300000 行

## 验证方式

本次没有修改作业表，只在 `test.public` 上执行只读核对：

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT
    u.city,
    count(DISTINCT o.id) AS paid_orders,
    sum(oi.quantity * oi.unit_price) AS revenue,
    avg(oi.quantity * oi.unit_price) AS avg_item_amount
FROM users u
JOIN orders o ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
WHERE o.status = 'paid'
  AND o.created_at >= now() - interval '90 days'
  AND p.category IN ('book', 'course')
GROUP BY u.city
HAVING sum(oi.quantity * oi.unit_price) > 50000
ORDER BY revenue DESC
LIMIT 10;
```

## 本次复核到的执行计划摘要

- 多次本地复核的 `Execution Time`：约 297-350 ms
- 多次本地复核的 `Planning Time`：约 2.6-11.1 ms
- `orders`：`Bitmap Heap Scan`，下层使用 `Bitmap Index Scan on idx_orders_status_created_user`
- `order_items`：`Seq Scan`
- `products`：`Seq Scan`，`Rows Removed by Filter: 500`
- `users`：`Seq Scan`
- JOIN：3 个 `Hash Join`
- 聚合：`GroupAggregate`，实际输出 5 个城市分组
- 排序：最终 `Sort Method: quicksort`
- Buffers：第一次复核出现 `shared hit=858 read=2592`，缓存预热后复跑出现 `shared hit=3450`

多次运行的核心计划节点一致，时间和 Buffers 变化主要说明缓存状态会影响单次运行指标。本次批改判断索引是否使用，主要看计划中是否出现具体的索引扫描节点。

## 逐题批改

### Q1. 扫描节点归属

你的回答：

- 节点属于 `users` 表。
- 估算行数是 `10000`。
- 实际行数是 `10000`。

判断：正确。

需要补上的理解：

- 不能把 `orders` 的 `100000` 行写到 `users` 这个节点上。
- 每个 plan node 的 `rows` 都只属于当前节点，不属于兄弟节点或别的表。
- `Seq Scan on public.users u ... rows=10000 ... actual ... rows=10000` 只能说明 `users` 这个扫描节点的估算和实际。

### Q2. 过滤条件归属

你的回答：

- `81772` 是 `orders` 表被过滤掉的行数。
- 它由 `o.status = 'paid'` 和 `o.created_at >= now() - interval '90 days'` 两个条件共同过滤出来。
- 不能拿它解释 `products.category`。

判断：正确。

这题已经比昨天稳定。继续保持一个规则：`Filter` 写在哪个扫描节点下面，`Rows Removed by Filter` 就归哪个节点。

### Q3. `products` 小表为什么可能 `Seq Scan`

你的回答：

- 选择度大概是 50%。
- 表数据量太小，即使有 `idx_products_category`，优化器也可能用 `Seq Scan`。
- 没用索引不一定是坏事，索引访问本身也有成本。

判断：正确。

补充一句更完整的表达：

> `products` 只有 1000 行，而且条件留下约 500 行，选择度不高；全表顺序扫再过滤可能比走索引再回表更便宜，所以 `Seq Scan` 是合理选择。

### Q4. 估算行数和实际行数

你的回答：

- `rows=18748` 是估算行数。
- `actual ... rows=18228` 是实际行数。
- 差距不大。
- 如果差距到数量级，可能影响执行计划选择。

判断：正确。

这题掌握得不错。注意以后要说得更精确一点：估算偏差会影响优化器对扫描方式、JOIN 顺序、JOIN 类型、聚合方式的成本判断。

### Q5. `HAVING` 到底过滤什么

你的回答中有两个错误：

- 你问“`HAVING` 先执行吗？”这里应修正为：`WHERE` 先于 `GROUP BY`，`HAVING` 在分组和聚合之后执行。
- 你写“过滤的是明细行”，这里应修正为：`HAVING` 过滤的是聚合后的组。

正确理解：`FROM / JOIN -> WHERE -> GROUP BY / 聚合 -> HAVING -> SELECT -> ORDER BY -> LIMIT`。

在这条 SQL 中：

- `WHERE` 先过滤订单状态、订单时间、商品分类。
- `GROUP BY u.city` 把剩余明细行按城市聚合。
- `HAVING sum(...) > 50000` 判断每个城市分组的收入总和。
- 如果某个城市收入不超过 50000，这整个城市分组会被过滤掉，而不是过滤某几条 `order_items` 明细行。

这题需要复训。

### Q6. 为什么是 `Hash Join`

你的回答：

- 等值连接是 `Hash Join` 的必要条件之一。
- 不能说“因为是等值连接，所以一定是 Hash Join”，因为等值连接也可能走其他 join 方法。
- `Nested Loop` 的条件你写了“小表驱动大表”。

判断：部分正确。

需要修正：

- “小表驱动大表”只说了一半。
- 更准确是：外层输入较小，并且内层表能通过 join key 或过滤条件做低成本查找时，`Nested Loop` 更可能划算。
- 如果外层小，但内层每次都要全表扫，大量循环仍然会很贵。

正确表达可以写成：

> `Hash Join` 常用于较大输入的等值连接。优化器也会比较 `Nested Loop` 和 `Merge Join`。如果外层结果很小，内层 join key 上有合适索引，并且每次查找成本低，优化器可能改用 `Nested Loop`。

### Q8. 加索引后怎么判断哪个索引起作用

你的回答：

- `idx_orders_status_created_user` 被明确使用了。
- 你写 `idx_order_items_order_id` 也“有”明显使用。
- 对“为什么不能只写加了索引，查询变快了”还不理解。

判断：第一点正确，第二点错误，第三点需要补。

本次实际计划中明确出现 `Bitmap Index Scan on idx_orders_status_created_user`。

所以可以说 `idx_orders_status_created_user` 被使用了。

但 `order_items` 的节点仍然是 `Seq Scan on public.order_items oi`。

因此不能说 `idx_order_items_order_id` 在这次计划里明显起作用。判断索引是否使用，要看计划里有没有对应的 `Index Scan`、`Index Only Scan`、`Bitmap Index Scan`，以及节点名称是否指向那个索引。

也不能只写“加了索引，查询变快了”，原因是：

- 这次只有 `orders` 的扫描方式明显变化。
- `order_items` 仍然扫描 300000 行。
- 3 个 JOIN 仍然是 `Hash Join`。
- 聚合和排序仍然存在。
- 执行时间受缓存、数据分布、重复运行次数影响，一次耗时不能单独证明索引让整条 SQL 变快。

## 对原练习末尾回答的补充批改

你写“为什么有些 SQL 会走全表扫描”：拿出大量列、数据量小、不值得走索引扫描。这个方向正确，还可以补上“条件选择度低”和“顺序读成本更低”。

你写“为什么有索引也不一定使用索引”：可能是选择度太低。正确。

你写“JOIN 顺序为什么会影响执行计划”：这里还没理解。JOIN 顺序影响中间结果大小和执行成本；对普通 `INNER JOIN` 来说，只要语义等价，最终结果通常不变，但执行成本可能差很多。

你写“行数估算为什么重要”：会影响执行计划生成。正确，但要具体到扫描方式、JOIN 顺序、JOIN 类型、聚合方式。

你写“`ANALYZE` 对优化器有什么作用”：收集可信的统计信息。正确。

你写“怎么对比加索引前后的执行计划”：主要看底层扫描是否用了索引。这个太窄。还要同时比较 JOIN 类型、JOIN 顺序、聚合方式、排序方式、`Execution Time`、`Planning Time`、Buffers、实际行数变化，以及哪些索引没有被使用。

## 当前掌握程度评价

本次评分：`72/100`。

比昨天进步明显的地方：

- 能稳定区分 `rows=...` 和 `actual ... rows=...`。
- 能把 `orders` 和 `products` 的过滤行数归到正确表。
- 能解释小表、低选择度时为什么可能不走索引。
- 对估算行数偏差为什么重要已经有初步判断。

仍需重点修正：

- `WHERE` 和 `HAVING` 的执行位置。
- `HAVING` 过滤的是分组，不是明细行。
- `Nested Loop` 不是简单的“小表驱动大表”，还要看内层是否有低成本访问路径。
- 判断索引是否使用必须以执行计划中的具体索引扫描节点为证据。
- 索引前后对比不能只看有没有索引，要看整棵执行计划和运行指标。

## 下一步练习建议

下一轮只练 3 件事：

1. 用一句话解释 `WHERE -> GROUP BY -> HAVING`，并说明这条 SQL 里每一步过滤的对象。
2. 从实际执行计划里列出每个索引是否被使用，必须引用对应节点名称。
3. 重写索引前后对比表，把扫描方式、JOIN、聚合、排序、时间、Buffers、索引使用情况都写进去。
