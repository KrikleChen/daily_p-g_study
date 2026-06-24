# PostgreSQL and Greenplum Kernel Learning Roadmap

目标不是只会使用 PostgreSQL，而是逐步做到：

- 完整理解 PostgreSQL 的使用、运维、执行计划和内核主流程。
- 能读 PostgreSQL 源码，理解 SQL 从客户端进入到返回结果的完整路径。
- 能开发 PostgreSQL extension，包括 SQL extension、C function、hook、background worker 和 FDW。
- 理解 Greenplum / GP 这类 PostgreSQL 衍生 MPP 数据库的架构、执行模型和调优重点。
- 长期具备阅读、修改、验证数据库内核代码的能力。

## 学习原则

- 先吃透单机 PostgreSQL，再看 Greenplum。
- 每个主题都按 `文档 -> 实验 -> 源码 -> 记录` 推进。
- 不只看文章，必须用 SQL、`psql`、`EXPLAIN ANALYZE`、日志和源码验证。
- 每天至少记录一个明确问题、一个实验现象和一个结论。
- 先建立主流程地图，再深入局部模块。

## 阶段 0：基础能力

目标：具备阅读和调试数据库内核代码的基础。

需要补齐：

- C 语言：指针、内存管理、结构体、宏、函数指针、动态库。
- Linux：进程、信号、共享内存、文件系统、权限、网络。
- 工具链：Makefile、autoconf、gcc/clang、gdb/lldb、perf、strace/dtrace。
- 数据结构：hash table、list、tree、queue、bitmap。
- 数据库基础：B+Tree、WAL、MVCC、事务、锁、buffer pool、执行计划。

建议产出：

- 能从源码编译 PostgreSQL。
- 能启动一个调试版 PostgreSQL 实例。
- 能用调试器 attach 到 backend process。

## 阶段 1：PostgreSQL 使用和 DBA 基础

目标：熟练使用 PostgreSQL，知道数据库日常怎么运行、怎么观察、怎么维护。

重点主题：

- `psql` 基本使用。
- SQL、DDL、DML、DCL。
- database、schema、table、index、view、sequence。
- role、privilege、ownership。
- 事务、隔离级别、锁等待。
- 常见索引和执行计划。
- backup / restore。
- `VACUUM`、`ANALYZE`、autovacuum。
- `pg_stat_activity`、`pg_stat_database`、`pg_stat_user_tables`、`pg_stat_statements`。

练习：

- 初始化一个本地 PostgreSQL 实例。
- 建库、建 schema、建表、建索引。
- 写 30 条 SQL，覆盖 join、aggregate、subquery、window function。
- 用两个 `psql` session 模拟事务隔离和锁等待。
- 对慢 SQL 使用 `EXPLAIN ANALYZE` 做前后对比。

官方文档入口：

- PostgreSQL Tutorial: https://www.postgresql.org/docs/current/tutorial.html
- SQL Language: https://www.postgresql.org/docs/current/sql.html
- Performance Tips: https://www.postgresql.org/docs/current/performance-tips.html
- Monitoring: https://www.postgresql.org/docs/current/monitoring.html
- Routine Maintenance: https://www.postgresql.org/docs/current/maintenance.html

## 阶段 2：PostgreSQL 内核主流程

目标：能讲清楚一条 SQL 从客户端进来到返回结果，中间经过哪些模块。

主线顺序：

1. 连接和进程模型：`postmaster`、backend process、shared memory。
2. SQL 处理流程：parser、analyzer、rewriter、planner、executor。
3. 存储系统：heap table、page、tuple、TOAST、visibility map、free space map。
4. 索引系统：btree、hash、gin、gist、brin。
5. 事务系统：transaction id、snapshot、MVCC、pg_xact、subtransaction。
6. WAL 和恢复：WAL record、checkpoint、crash recovery、replication。
7. buffer manager：shared buffers、dirty page、replacement、checkpoint。
8. lock manager：heavyweight lock、lightweight lock、spinlock、deadlock detector。
9. optimizer：path、plan、cost model、statistics、join order。
10. executor：seq scan、index scan、join、aggregate、sort、materialize。

源码目录重点：

```text
src/backend/parser
src/backend/rewrite
src/backend/optimizer
src/backend/executor
src/backend/access/heap
src/backend/access/nbtree
src/backend/access/transam
src/backend/storage/buffer
src/backend/storage/lmgr
src/backend/catalog
src/include
```

建议第一条源码阅读主线：

```text
客户端发送 SELECT
-> backend 接收 SQL
-> raw parser
-> parse analysis
-> rewrite
-> planner
-> executor
-> heap/index access
-> 返回结果
```

每天记录时要写清楚：

- 今天读的是哪个模块。
- 入口函数是什么。
- 调用链是什么。
- 用什么 SQL 触发。
- 观察到什么日志、断点或执行计划。

## 阶段 3：PostgreSQL Extension 开发

目标：能写 PostgreSQL extension，而不是只会写 SQL 函数。

学习顺序：

1. SQL-only extension：`.control` 文件和 `--1.0.sql` 文件。
2. C function：用 PGXS 编译共享库。
3. 自定义类型：input/output function、operator、cast。
4. aggregate：transition function、final function。
5. hook：planner hook、executor hook、ProcessUtility hook。
6. background worker：写 PostgreSQL 后台进程。
7. FDW：访问外部数据源。
8. index access method / table access method：高阶扩展点。

官方文档入口：

- Server Programming: https://www.postgresql.org/docs/current/server-programming.html
- Extending SQL: https://www.postgresql.org/docs/current/extend.html
- CREATE EXTENSION: https://www.postgresql.org/docs/current/sql-createextension.html
- PGXS: https://www.postgresql.org/docs/current/extend-pgxs.html
- C-Language Functions: https://www.postgresql.org/docs/current/xfunc-c.html

第一个插件建议：

- 名称：`pg_hello`
- 功能：`select pg_hello();`
- 返回 PostgreSQL 版本、当前数据库、当前用户。
- 目标：完整走通编译、安装、`CREATE EXTENSION`、调用、卸载。

第二个插件建议：

- 名称：`pg_query_logger`
- 功能：用 hook 记录 SQL 执行时间。
- 目标：理解 hook 注册、生命周期、日志输出和异常处理。

## 阶段 4：Greenplum / GP 和 MPP 架构

目标：理解 PostgreSQL 如何演化成 MPP shared-nothing 分布式数据库。

重点概念：

- shared-nothing 架构。
- coordinator / master。
- standby master。
- segment。
- primary segment / mirror segment。
- QD：Query Dispatcher。
- QE：Query Executor。
- interconnect。
- data distribution。
- distribution key。
- data skew。
- motion node。
- distributed transaction。
- resource group / resource queue。
- append-optimized table。
- partition。
- GPORCA optimizer。

学习顺序：

1. 先理解 Greenplum 集群拓扑。
2. 再理解表数据如何分布到 segments。
3. 然后看一条 SQL 如何由 QD 分发到 QE。
4. 接着重点看 Motion 节点和数据重分布。
5. 最后研究资源管理、分布式事务和 ORCA 优化器。

官方文档入口：

- Greenplum 7 Documentation: https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/landing-index.html
- Greenplum Architecture: https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/admin_guide-intro-arch_overview.html
- Query Processing: https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/admin_guide-query-topics-parallel-proc.html
- Distribution and Skew: https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/admin_guide-distribution.html
- Apache Cloudberry: https://cloudberry.apache.org/

## PostgreSQL 和 Greenplum 对比主线

| 主题 | PostgreSQL | Greenplum |
| --- | --- | --- |
| 架构 | 单机数据库 | MPP shared-nothing |
| 数据位置 | 本机 heap/index | 分布在 segments |
| 查询入口 | backend process | coordinator / QD |
| 执行方式 | 本地 executor | QD 分发，QE 并行执行 |
| 优化器 | PG planner | GP planner / ORCA |
| JOIN | 本地 join | 可能需要 Motion |
| 事务 | 单机 MVCC | 分布式事务 |
| 性能核心 | 索引、缓存、执行计划 | 分布键、倾斜、Motion、并行度 |

## 时间规划

- 第 1-2 个月：PostgreSQL 使用、SQL、索引、事务、运维。
- 第 3-5 个月：PostgreSQL 源码主流程，能讲清楚一条 SQL 的生命周期。
- 第 6-7 个月：写 extension，覆盖 SQL extension、C extension、hook。
- 第 8-10 个月：Greenplum 架构、分布式执行、Motion、数据倾斜。
- 第 11-12 个月：读 GP / Cloudberry 源码，做小修改。
- 1 年以后：深入优化器、存储、WAL、分布式事务、插件框架。

## 每日记录模板

```markdown
# YYYY-MM-DD

## 今日主题

## 要回答的问题

## 实验环境

## SQL / 命令

## 观察到的现象

## 相关源码文件

## 源码理解

## PG 和 GP 的差异

## 结论

## 明天继续的问题
```

## 第一周任务

1. 本地安装或源码编译 PostgreSQL。
2. 用 `initdb` 初始化一个实例。
3. 用 `pg_ctl` 启停实例。
4. 用 `psql` 建库建表。
5. 开两个 session 测试事务隔离。
6. 对一条 SQL 跑 `EXPLAIN ANALYZE`。
7. 写第一篇正式学习记录：一条 `SELECT` 在 PostgreSQL 中大概经历什么流程。

## 长期产出

- 每天一篇学习记录。
- 每周一篇主题总结。
- 每月完成一个实验项目。
- 每个源码模块都保留入口函数、调用链、实验 SQL 和结论。
- 至少完成两个 PostgreSQL extension。
- 至少完成一次 PostgreSQL 和 Greenplum 同主题对比实验。
