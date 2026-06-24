# 练习：用 EXPLAIN ANALYZE 分析订单收入统计 SQL

## 学习目标

这道题的目标不是看 SQL 能不能查出结果，而是练习读懂 PostgreSQL 的执行计划。

通过 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 分析一条稍微复杂的 SQL，理解 PostgreSQL 在执行查询时做了哪些事情。

这条 SQL 包含：

- 多表 `JOIN`
- 时间范围过滤
- 状态过滤
- 商品分类过滤
- `GROUP BY`
- `HAVING`
- `ORDER BY`
- `LIMIT`
- 聚合函数
- `count(DISTINCT ...)`

## 需要回答的问题

跑完这道题以后，要能回答这些问题：

- 哪些表用了 `Seq Scan`？
- 哪些表用了 `Index Scan` 或 `Bitmap Heap Scan`？
- PostgreSQL 选择了哪种 JOIN：`Hash Join`、`Nested Loop` 还是 `Merge Join`？
- 哪个执行节点最耗时？
- 估算行数和实际行数差距大不大？
- `Planning Time` 和 `Execution Time` 分别是什么意思？
- `Buffers: shared hit` 和 `Buffers: shared read` 分别说明什么？
- 加索引以后，执行计划有没有变化？
- 哪个索引真正起了作用？

## 第一步：准备测试数据

在一个测试数据库里执行下面的 SQL。

```sql
DROP TABLE IF EXISTS order_items, orders, products, users CASCADE;

CREATE TABLE users (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL,
    city text NOT NULL
);

CREATE TABLE products (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL,
    category text NOT NULL,
    price numeric(10,2) NOT NULL
);

CREATE TABLE orders (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id bigint NOT NULL REFERENCES users(id),
    status text NOT NULL,
    created_at timestamp NOT NULL
);

CREATE TABLE order_items (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id bigint NOT NULL REFERENCES orders(id),
    product_id bigint NOT NULL REFERENCES products(id),
    quantity int NOT NULL,
    unit_price numeric(10,2) NOT NULL
);

INSERT INTO users (name, city)
SELECT
    'user_' || g,
    CASE g % 5
        WHEN 0 THEN 'beijing'
        WHEN 1 THEN 'shanghai'
        WHEN 2 THEN 'shenzhen'
        WHEN 3 THEN 'hangzhou'
        ELSE 'guangzhou'
    END
FROM generate_series(1, 10000) AS g;

INSERT INTO products (name, category, price)
SELECT
    'product_' || g,
    CASE g % 4
        WHEN 0 THEN 'book'
        WHEN 1 THEN 'course'
        WHEN 2 THEN 'hardware'
        ELSE 'software'
    END,
    round((random() * 500 + 10)::numeric, 2)
FROM generate_series(1, 1000) AS g;

INSERT INTO orders (user_id, status, created_at)
SELECT
    (random() * 9999 + 1)::bigint,
    CASE WHEN random() < 0.75 THEN 'paid' ELSE 'cancelled' END,
    now() - ((random() * 365)::int || ' days')::interval
FROM generate_series(1, 100000);

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT
    (random() * 99999 + 1)::bigint,
    (random() * 999 + 1)::bigint,
    (random() * 5 + 1)::int,
    round((random() * 500 + 10)::numeric, 2)
FROM generate_series(1, 300000);

ANALYZE;
```

数据规模：

- `users`：1 万行
- `products`：1000 行
- `orders`：10 万行
- `order_items`：30 万行

这里执行 `ANALYZE` 是为了让 PostgreSQL 收集统计信息。优化器会根据统计信息估算每一步大概会处理多少行。

## 第二步：先不加额外索引，分析查询

先直接执行下面的 SQL。

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

## 这条 SQL 在查什么

这条 SQL 的业务含义是：

统计最近 90 天内，已支付订单中，商品分类为 `book` 或 `course` 的城市收入排行。

输出字段：

- `city`：城市
- `paid_orders`：已支付订单数量
- `revenue`：总收入
- `avg_item_amount`：订单明细平均金额

## 第三步：从下往上读执行计划

执行计划要从最里面、最下面的节点开始读。

### 1. 看扫描方式

记录每张表是怎么被扫描的：

- `users`
- `orders`
- `order_items`
- `products`

重点观察这些节点：

- `Seq Scan`：顺序扫描，也就是全表扫描
- `Index Scan`：索引扫描
- `Index Only Scan`：只通过索引就能返回结果
- `Bitmap Index Scan`：先通过索引生成 bitmap
- `Bitmap Heap Scan`：再根据 bitmap 回表读取数据

笔记里可以这样写：

| 表 | 扫描方式 | 估算行数 | 实际行数 | 备注 |
| --- | --- | --- | --- | --- |
| `users` |  |  |  |  |
| `orders` |  |  |  |  |
| `order_items` |  |  |  |  |
| `products` |  |  |  |  |

### 2. 看过滤条件

重点找这些条件：

- `o.status = 'paid'`
- `o.created_at >= now() - interval '90 days'`
- `p.category IN ('book', 'course')`

需要记录：

- `Rows Removed by Filter`
- 估算 `rows`
- 实际 `rows`

如果 `Rows Removed by Filter` 很大，说明 PostgreSQL 读了很多行，但过滤掉了很多行。

### 3. 看 JOIN 类型

重点看 PostgreSQL 选择了哪种 JOIN：

- `Hash Join`
- `Nested Loop`
- `Merge Join`

每个 JOIN 都记录：

- 左边输入是什么
- 右边输入是什么
- JOIN 条件是什么
- 估算输出多少行
- 实际输出多少行

笔记里可以这样写：

| JOIN 节点 | JOIN 类型 | JOIN 条件 | 估算行数 | 实际行数 | 我的理解 |
| --- | --- | --- | --- | --- | --- |
| 1 |  |  |  |  |  |
| 2 |  |  |  |  |  |
| 3 |  |  |  |  |  |

### 4. 看聚合节点

重点看有没有这些节点：

- `HashAggregate`
- `GroupAggregate`

记录：

- 聚合前有多少行
- 聚合后有多少组
- `GROUP BY u.city` 最后产生了几个城市分组
- `HAVING` 有没有过滤掉某些分组

### 5. 看排序和 LIMIT

重点看有没有：

- `Sort`
- `top-N heapsort`
- `Limit`

如果有 `LIMIT 10`，PostgreSQL 可能不需要完整排序所有结果，而是只维护前 10 条。

### 6. 看时间和 Buffers

记录这些信息：

- `Planning Time`
- `Execution Time`
- `Buffers: shared hit`
- `Buffers: shared read`

理解方式：

- `Planning Time`：优化器生成执行计划花了多久
- `Execution Time`：真正执行 SQL 花了多久
- `shared hit`：从 PostgreSQL shared buffers 里命中的页
- `shared read`：需要从磁盘或操作系统缓存读取的页

## 第四步：添加索引再对比

保存第一版执行计划以后，再创建下面这些索引。

```sql
CREATE INDEX idx_orders_status_created_user
ON orders (status, created_at, user_id);

CREATE INDEX idx_order_items_order_id
ON order_items (order_id);

CREATE INDEX idx_order_items_product_id
ON order_items (product_id);

CREATE INDEX idx_products_category
ON products (category);

ANALYZE;
```

然后重新执行同一条 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 查询。

## 对比表

跑完加索引前后两版计划以后，填写这张表。

| 对比项 | 加索引前 | 加索引后 | 我的理解 |
| --- | --- | --- | --- |
| `Execution Time` |  |  |  |
| `Planning Time` |  |  |  |
| `orders` 扫描方式 |  |  |  |
| `order_items` 扫描方式 |  |  |  |
| `products` 扫描方式 |  |  |  |
| 主要 JOIN 类型 |  |  |  |
| 聚合方式 |  |  |  |
| `shared hit` |  |  |  |
| `shared read` |  |  |  |

## 每日笔记模板

可以把下面这段复制到当天的学习笔记里。

```markdown
# YYYY-MM-DD EXPLAIN ANALYZE 练习：订单收入统计 SQL

## 今日主题

分析一条多表 JOIN + 聚合 SQL 的 PostgreSQL 执行计划。

## 今天要回答的问题

PostgreSQL 会如何执行这条订单收入统计 SQL？加索引前后执行计划有什么变化？

## 实验环境

- PostgreSQL 版本：
- 安装方式：
- 数据库名：
- 数据量：

## SQL 目的

统计最近 90 天内，已支付订单中，商品分类为 book 或 course 的城市收入排行。

## 加索引前的执行计划

粘贴完整的 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 输出。

## 加索引后的执行计划

粘贴完整的 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 输出。

## 观察记录

- 扫描节点：
- JOIN 节点：
- 聚合节点：
- 排序和 LIMIT：
- 最耗时的节点：
- 估算行数和实际行数：
- Buffers：

## 我的理解

用自己的话解释 PostgreSQL 是怎么执行这条 SQL 的。

## 初步结论

不要只写“加索引以后变快了”，要写清楚：

- 哪些扫描方式变了
- 哪些 JOIN 方式变了
- 哪个索引被使用了
- 哪个索引没有明显作用
- 总执行时间变化了多少
- Buffers 有没有减少

## 明天继续的问题

- 
```

## 这道题最终要学会什么

做完这道题以后，至少要能说清楚：

- 为什么有些 SQL 会走全表扫描
- 为什么有索引也不一定使用索引
- JOIN 顺序为什么会影响执行计划
- 行数估算为什么重要
- `ANALYZE` 对优化器有什么作用
- 怎么对比加索引前后的执行计划
