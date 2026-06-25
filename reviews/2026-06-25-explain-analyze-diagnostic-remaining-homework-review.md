# 2026-06-25 EXPLAIN ANALYZE 诊断题剩余题目批改记录

## 批改对象

- 诊断题记录：`reviews/2026-06-24-explain-analyze-diagnostic-drill.md`
- 原练习文件：`exercises/explain-analyze-order-revenue.md`
- 本次范围：Q7-Q12
- 主题：JOIN 顺序、索引证据、`Bitmap Index Scan` / `Bitmap Heap Scan`、Buffers、`top-N heapsort`、索引前后对比

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

本次只做只读验证：

- 用中间结果计数 SQL 复核 JOIN 前后的行数变化。
- 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 复核实际计划节点。
- 没有修改 `public` schema 表和索引。

关键复核结果：

| 阶段 | 行数 |
| --- | ---: |
| `orders` 总行数 | 100000 |
| 过滤后的 `orders` | 18228 |
| `products` 总行数 | 1000 |
| 过滤后的 `products` | 500 |
| `order_items` 总行数 | 300000 |
| `order_items` join 过滤后的 `orders` | 54647 |
| `order_items` join 过滤后的 `products` | 150035 |
| `order_items` 同时 join 两个过滤结果 | 27326 |
| `GROUP BY` 前最终输入行数 | 27326 |

执行计划关键节点：

- `orders`：`Bitmap Heap Scan`，下层是 `Bitmap Index Scan on idx_orders_status_created_user`
- `order_items`：`Seq Scan`
- `products`：`Seq Scan`
- `users`：`Seq Scan`
- JOIN：3 个 `Hash Join`
- 聚合：`GroupAggregate`
- 排序：`Sort Method: quicksort`
- Buffers：缓存预热后可见 `shared hit=3450`

## Q7. JOIN 顺序为什么重要

你的回答：

- “没理解，你能给我讲清楚全流程吗”

判断：未完成作答，但这个问题问得对，说明你卡在“SQL 文本顺序、逻辑顺序、执行计划顺序”的区别上。

正确答案：

在这道题里，`WHERE` 里有两个只依赖单表的过滤：

- `orders` 上的 `status = 'paid'` 和 `created_at >= now() - interval '90 days'`
- `products` 上的 `category IN ('book', 'course')`

优化器可以把这些条件下推到表扫描阶段。实际效果是：

- `orders` 从 100000 行先缩到 18228 行。
- `products` 从 1000 行先缩到 500 行。
- 再拿过滤后的结果去 join `order_items`，最终 `GROUP BY` 前是 27326 行。

需要特别注意：

- 从 `order_items` 角度看，300000 行被过滤和 JOIN 限制后变成 27326 行，中间结果变小。
- 从 `orders` 角度看，18228 个订单 join `order_items` 后变成 54647 行，这是因为一个订单可以有多条订单明细。
- 所以不能简单说 JOIN 一定变小或一定变大；要看从哪张表、哪一步观察。

如果先让 `order_items` 这张 300000 行的大表参与大量 JOIN，再晚一点过滤，最终结果可能还是 27326 行，但前面的 JOIN、排序、聚合要处理更多中间数据，成本可能更高。

对普通等价 `INNER JOIN` 来说，JOIN 顺序通常不改变最终结果，但会改变中间结果大小和执行成本。

本题需要复训。

## Q8. 加索引后怎么判断哪个索引起作用

你的回答：

- `idx_orders_status_created_user` 被明确使用了。
- 你认为 `idx_order_items_order_id` 也能看出明显使用。
- 对“为什么不能只写加了索引，查询变快了”不理解。

判断：部分正确。

正确点：

- `idx_orders_status_created_user` 的确被明确使用，因为计划里出现了 `Bitmap Index Scan on idx_orders_status_created_user`。

需要修正：

- `idx_order_items_order_id` 没有从这份计划里看出明显使用。
- 因为 `order_items` 的节点是 `Seq Scan on public.order_items oi`，不是 `Index Scan`，也不是 `Bitmap Index Scan`。

判断索引是否使用，要看计划中是否出现具体索引名和索引扫描节点。例如：

- `Index Scan using ...`
- `Index Only Scan using ...`
- `Bitmap Index Scan on ...`

不能只因为建过索引，就说它在当前 SQL 里起作用。

## Q9. `Bitmap Index Scan` 和 `Bitmap Heap Scan`

你的回答：

- `Bitmap Index Scan`：扫索引对比。
- `Bitmap Heap Scan`：对比桶内的内容。
- 普通 `Index Scan` 为什么不一定使用：未答。

判断：方向不够准确，需要重学。

正确答案：

`Bitmap Index Scan` 主要做的是：

- 扫描索引。
- 找到满足索引条件的行位置，也就是 heap tuple 的位置。
- 把这些位置组织成一个 bitmap。

在本题里，它用的是 `idx_orders_status_created_user`，条件是：

- `status = 'paid'`
- `created_at >= now() - interval '90 days'`

`Bitmap Heap Scan` 主要做的是：

- 根据上一步 bitmap 里记录的 heap page / tuple 位置，回到表数据页读取真实行。
- 对必要条件做 recheck。
- 返回真正需要的列，例如 `orders.id` 和 `orders.user_id`。

为什么不一定直接用普通 `Index Scan`：

- 普通 `Index Scan` 更像是按索引条目一条一条去表里取数据。
- 如果命中行比较多，可能产生很多随机访问。
- `Bitmap Heap Scan` 可以先把要访问的表页合并起来，再按表页读取，减少重复和随机访问成本。

不要把这里的 bitmap 理解成 `Hash Join` 的桶。`Bitmap Heap Scan` 不是“对比桶内内容”，它是“按 bitmap 记录的位置回表取数据”。

## Q10. `Buffers` 的含义

你的回答：未答。

判断：未完成。

正确答案：

如果看到 `Buffers: shared hit=3450 read=0`：

- `shared hit=3450` 表示 PostgreSQL 在 shared buffers 里命中了 3450 个数据页。
- `shared read=0` 表示这次执行没有从 PostgreSQL 视角把新的数据页读入 shared buffers。
- `read=0` 不能直接说明这条 SQL 没有任何 IO 成本。

原因：

- 它仍然访问了 shared buffers，这是逻辑 IO。
- 它仍然有 CPU 成本、JOIN 成本、排序成本、聚合成本。
- 数据页可能是前面执行时已经读入缓存的，所以这次 `read=0` 只能说明缓存已经热了。
- `shared read` 也不完全等同于真实物理磁盘读，因为操作系统缓存也可能参与。

更准确表达：

> `read=0` 说明这次执行在 PostgreSQL shared buffers 层没有发生新的 shared read，但不能说明查询没有成本，也不能说明系统完全没有 IO 压力。

## Q11. `ORDER BY ... LIMIT` 和 `top-N heapsort`

你的回答：未答。

判断：未完成。

正确答案：

这条 SQL 有：

```sql
ORDER BY revenue DESC
LIMIT 10;
```

理论上，`ORDER BY ... LIMIT N` 可能触发 `top-N heapsort`。它的作用是：

- 不完整排序所有行。
- 只维护当前最靠前的 N 行。
- 适合“输入很多行，但只要前 N 行”的场景。

但这道题里聚合后实际只有 5 个城市分组，而 `LIMIT 10` 要 10 行。

所以 PostgreSQL 没必要维护 top 10，因为总共只有 5 行。实际计划里看到的是：

- `Sort Method: quicksort`
- `Memory: 25kB`

这说明最终排序数据很小。聚合后只有 5 行时，最终 `ORDER BY revenue DESC LIMIT 10` 不是主要成本；主要成本在扫描 `order_items`、JOIN、聚合前排序和聚合。

## Q12. 如何做一次合格的索引前后对比

你的回答：未答。

判断：未完成。

参考答案：

一张合格的对比表至少要这样写：

| 对比项 | 加索引前 | 加索引后 | 结论 |
| --- | --- | --- | --- |
| `orders` 扫描方式 | `Seq Scan` | `Bitmap Heap Scan + Bitmap Index Scan` | `idx_orders_status_created_user` 起作用 |
| `order_items` 扫描方式 | `Seq Scan` | `Seq Scan` | `idx_order_items_order_id` 没有明显用于当前计划 |
| `products` 扫描方式 | `Seq Scan` | `Seq Scan` | 表小且选择度约 50%，全表扫合理 |
| `users` 扫描方式 | `Seq Scan` | `Seq Scan` | 小表全表扫后 hash，合理 |
| JOIN 类型 | `Hash Join` | `Hash Join` | JOIN 策略没有明显变化 |
| 聚合方式 | `GroupAggregate` | `GroupAggregate` | 聚合策略没有明显变化 |
| 排序方式 | 以实际计划为准 | `quicksort` | 最终排序行数很少，不是主要成本 |
| `Execution Time` | 以实际计划为准 | 多次约 297-350 ms | 不能只看一次时间，要考虑缓存 |
| `Planning Time` | 以实际计划为准 | 多次约 2.6-11.1 ms | 建索引后可选路径更多，规划时间可能变化 |
| Buffers | 以实际计划为准 | 热缓存时 `shared hit=3450` | Buffers 要结合缓存状态解释 |
| 明确使用的索引 | 无或以实际计划为准 | `idx_orders_status_created_user` | 以计划中的索引扫描节点为证据 |
| 没有明显使用的索引 | 以实际计划为准 | `idx_order_items_order_id`、`idx_order_items_product_id`、`idx_products_category` | 建了不等于当前查询用了 |

合格结论应该这样写：

> 加索引后，`orders` 的过滤条件使用了组合索引 `idx_orders_status_created_user`，扫描方式从 `Seq Scan` 变成 `Bitmap Heap Scan + Bitmap Index Scan`。但 `order_items`、`products`、`users` 仍然是 `Seq Scan`，JOIN 仍然是 `Hash Join`，聚合仍然是 `GroupAggregate`。因此只能说这个索引改变了 `orders` 的访问方式，不能简单说“加了索引所以整条 SQL 一定更快”。整体耗时还要结合中间行数、Buffers、缓存状态和多次执行结果判断。

## 本次综合评价

本次只看 Q7-Q12，评分：`38/100`。

这个分数不是说你完全不会，而是因为 Q10-Q12 没写，Q9 的核心概念写错，Q7 还停在“没理解”。按作业完整性评分会比较低。

已经有一点基础的地方：

- Q8 能识别 `idx_orders_status_created_user` 被使用。
- 你能主动指出 Q7 没理解，并追问“过滤顺序”这个关键问题。

需要重点补的地方：

- `Bitmap Index Scan` 和 `Bitmap Heap Scan` 的分工。
- `Buffers` 不能只按“有没有读磁盘”粗略理解。
- `top-N heapsort` 是 `ORDER BY ... LIMIT` 的排序优化，不是必然出现。
- 索引对比必须看整棵执行计划，不是只看某个索引是否出现。
- JOIN 顺序主要影响中间结果大小和执行成本，等价 `INNER JOIN` 的最终结果通常不变。

## 下一步练习建议

下一步不要再扩展新知识，先把 Q7-Q12 改写一遍。

建议你按这个顺序重答：

1. 用自己的话解释 `Bitmap Index Scan` 和 `Bitmap Heap Scan`。
2. 解释为什么 `idx_order_items_order_id` 没有在当前计划中明显使用。
3. 用 `300000 -> 54647 -> 27326` 解释 JOIN 顺序为什么影响成本。
4. 解释 `shared hit=3450 read=0` 不能说明 SQL 没有成本。
5. 写出一张完整的索引前后对比表。
