# Postgres 9.6 新特性之并行查询

在这一集中，我们将研究 PostgreSQL 9.6 中的一个新特性，并行查询。并行查询提供顺序扫描、连接和聚合的并行执行。

要查看此功能的实际效果，让我们先创建一个测试数据库。为了能充分的体现性能差异，我们需要大量数据。我制作了一个财务分类账，上面有 5000 万美元的随机金额，并且可以追溯到十年前；初始化测试数据库的脚本在后面的演示示例中。

接下来，我们将编写一个使用顺序扫描的只读查询。聚合函数是很好的候选者。我们将使用 `sum` 将分类帐中的金额相加，并执行 `EXPLAIN ANALYZE` 以进行基准测试和解释。

```sql
EXPLAIN ANALYZE SELECT sum(amount) FROM ledger;
```
```
                                                         QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=942773.40..942773.42 rows=1 width=32) (actual time=8614.531..8614.531 rows=1 loops=1)
   ->  Seq Scan on ledger  (cost=0.00..817778.52 rows=49997952 width=8) (actual time=0.022..2551.421 rows=50000000 loops=1)
 Planning time: 0.028 ms
 Execution time: 8614.548 ms
(4 rows)

Time: 8614.785 ms
```

阅读输出，我们可以看到 PostgreSQL 已选择按顺序运行此查询。太好了，正是我们需要的。默认情况下不启用并行查询。要打开它们，我们需要增加一个名为 `max_parallel_workers_per_gather` 的配置参数。

```sql
SHOW max_parallel_workers_per_gather;
```
```
 max_parallel_workers_per_gather
---------------------------------
 0
(1 row)

Time: 0.111 ms
```

让我们把它提高到四个，这恰好是这个工作站上的核心数。

```sql
SET max_parallel_workers_per_gather TO 4;
```

再次执行查询，我们可以看到 PostgreSQL 现在选择并行查询。它的速度大约快了四倍。

```sql
EXPLAIN ANALYZE SELECT sum(amount) FROM ledger;
```
```
                                                                   QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=475043.04..475043.05 rows=1 width=32) (actual time=2475.135..2475.135 rows=1 loops=1)
   ->  Gather  (cost=475042.60..475043.01 rows=4 width=32) (actual time=2475.117..2475.130 rows=5 loops=1)
         Workers Planned: 4
		 Workers Launched: 4
		 ->  Partial Aggregate  (cost=474042.60..474042.61 rows=1 width=32) (actual time=2473.571..2473.572 rows=1 loops=5)
		       -> Parallel Seq Scan on ledger  (cost=0.00..442793.88 rows=12499488 width=8) (actual time=0.015..1023.268 rows=1000000 loops=5)
 Planning time: 0.035 ms
 Execution time: 2477.926 ms
(8 rows)

Time: 2478.152 ms
```

总而言之，对于 PostgreSQL 9.6 上的顺序扫描，打开此功能可以享受速度提升。正如我之前提到的，只有顺序扫描可以并行化。让我们通过向表添加索引来创建此功能不可用的场景。索引列改变了一些东西，我们将看到：

```sql
CREATE INDEX ON ledger(date);
```

对于查询中的索引列，我们不能使用并行性：

```sql
EXPLAIN ANALYZE SELECT sum(amount) FROM ledger WHERE date = (current_date - 1);
```
```
 Aggregate  (cost=44977.22..44977.23 rows=1 width=32) (actual time=46.570..46.570 rows=1 loops=1)
   ->  Bitmap Heap Scan on ledger  (cost=252.24..44943.78 rows=13376 width=8) (actual time=3.107..44.317 rows=13736 loops=1)
         Recheck Cond: (date = (('now'::cstring)::date - 1))
		 Heap Blocks: exact=13433
		 ->  Bitmap Index Scan on ledger_date_idx  (cost=0.00..248.89 rows=13376 width=0) (actual time=1.566..1.566 rows=13736 loops=1)
		       Index Cond: (date = (('now'::cstring)::date - 1))
 Planning time: 0.150 ms
 Execution time: 46.591 ms
(8 rows)

Time: 47.122 ms
```

要永久启用此功能，请将参数添加到 PostgreSQL 配置文件中。


## 演示示例

```sql
CREATE TABLE ledger (
  id serial primary key,
  date date not null,
  amount decimal(12,2) not null
);

INSERT INTO ledger (date, amount)
  SELECT
    current_date - (random() * 3650)::integer,
	(random() * 1000000)::decimal(12,2) - 50000
  FROM
    generate_series(1, 50000000);
```

## 参考

[1] https://www.postgresql.org/docs/9.6/static/release-9-6.html

[2] https://www.postgresql.org/docs/current/runtime-config-resource.html

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第二十一集 [New in Postgres 9.6: Parallel Queries](https://www.pgcasts.com/episodes/new-in-postgres-9-6-parallel-queries)。

[2] 在新版本中，[并行查询默认是开启的](https://www.postgresql.org/docs/current/runtime-config-resource.html)，`max_parallel_workers_per_gather` 设置为 `2`。
