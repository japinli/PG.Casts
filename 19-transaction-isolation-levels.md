# 事务隔离级别

我们这一集的主题是事务隔离级别。事务隔离级别决定了并发事务对彼此的影响。 PostgreSQL 实现了三个事务隔离级别：读已提交（Read Committed）、可重复读（Repeatable Read）和可序列化（Serializable）。

我们将使用一个带有运行统计的简单分类帐表作为示例。

```sql
CREATE TABLE entry (
  id serial primary key,
  amount integer not null,
  running_total integer
);
```

默认的隔离级别，读已提交，保证未提交的更改对其它事务是不可见的。

```sql
-- conn 1
BEGIN;

INSERT INTO entry(amount) VALUES (10) RETURNING *;
```

通常，运行统计将在插入后触发器中处理，但为了使示例更易于理解，我们将其内联进行。`RETURN *` 返回插入的行，以便我们可以获取更新的 `id`。

```sql
-- conn 1
UPDATE entry
  SET running_total = (
    SELECT sum(amount) FROM entry)
WHERE id = 1
RETURNING *;
```
```
 id | amount | running_total
----+--------+---------------
  1 |     10 |            10
(1 row)

UPDATE 1
```

现在在我们提交之前，让我们看一下来自另一个连接的 `entry` 表。

```sql
-- conn 2
SELECT * FROM entry;
```
```
 id | amount | running_total
----+--------+---------------
(0 rows)
```

在我们提交之前，我们的更改是不可见的。

```sql
-- conn 1
COMMIT;
```

```sql
-- conn 2
SELECT * FROM entry;
```
```
 id | amount | running_total
----+--------+---------------
  1 |     10 |            10
(1 row)
```

但是，如果这两个同时发生呢？

```sql
-- conn 1
BEGIN;

INSERT INTO entry(amount) VALUES (10) returning *;
```
```
 id | amount | running_total
----+--------+---------------
  2 |     10 |
(1 row)

INSERT 0 1
```
```sql
-- conn 1
UPDATE ENTRY
  SET running_total = (
    SELECT sum(amount) FROM entry)
WHERE id = 2
RETURNING *;
```
```
 id | amount | running_total
----+--------+---------------
  2 |     10 |            20
(1 row)

UPDATE 1
```

```sql
-- conn 2
BEGIN;

INSERT INTO entry(amount) VALUES (10) returning *;
```
```
 id | amount | running_total
----+--------+---------------
  3 |     10 |
(1 row)

INSERT 0 1
```
```sql
-- conn 2
UPDATE ENTRY
  SET running_total = (
    SELECT sum(amount) FROM entry)
WHERE id = 3
RETURNING *;
```
```
 id | amount | running_total
----+--------+---------------
  3 |     10 |            20
(1 row)

UPDATE 1
```

```sql
-- conn 1
COMMIT;
```

```sql
-- conn 2
COMMIT;
```

```sql
-- conn 1 or conn 2
SELECT * FROM entry;
```
```
 id | amount | running_total
----+--------+---------------
  1 |     10 |            10
  2 |     10 |            20
  3 |     10 |            20
(3 rows)
```

即使这些语句在事务中运行，统计也是不正确。这是因为两个事务都看不到对方新插入的行。为了解决这个问题，我们将使用最强的隔离级别，可序列化。这个级别保证并发事务可以串行执行。

让我们重置表并再试一次。

```sql
TRUNCATE entry RESTART IDENTITY;
```

要更改隔离级别，我们在事务开始时使用 `SET TRANSACTION` 语句<sup>[1]</sup>。

```sql
-- conn 1
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

INSERT INTO entry(amount) VALUES (10) RETURNING *;
```
```
 id | amount | running_total
----+--------+---------------
  1 |     10 |
(1 row)

INSERT 0 1
```
```sql
-- conn 1
UPDATE ENTRY
  SET running_total = (
    SELECT sum(amount) FROM entry)
WHERE id = 1
RETURNING *;
```
```
 id | amount | running_total
----+--------+---------------
  1 |     10 |            10
(1 row)

UPDATE 1
```

我们还可以将隔离级别设置为 `BEGIN` 语句的一部分。

```sql
-- conn 2
BEGIN ISOLATION LEVEL SERIALIZABLE;

INSERT INTO entry(amount) VALUES (10) RETURNING *;
```
```
 id | amount | running_total
----+--------+---------------
  2 |     10 |
(1 row)

INSERT 0 1
```

```sql
-- conn 2
UPDATE ENTRY
  SET running_total = (
    SELECT sum(amount) FROM entry)
WHERE id = 2
RETURNING *;
```
```
 id | amount | running_total
----+--------+---------------
  2 |     10 |            10
(1 row)

UPDATE 1
```

```sql
-- conn 2
COMMIT;
```

第一个提交的事务将会成功。但是注意当我们提交另一个事务时会发生什么。

```sql
-- conn 1
COMMIT;
```
```
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
HINT:  The transaction might succeed if retried.
```

PostgreSQL 检测到这两个事务不可能连续执行并拒绝提交。使用可序列化隔离级别的应用程序必须准备好在提交时处理序列化错误。更高的隔离级别可能会降低性能，但它们可以防止数据异常。更多关于可重复读隔离级别、默认和可序列化隔离级别之间的中间立场，可以在 PostgreSQL 文档中找到。


## 译者著

[1] 在 `BEGIN` 语句和 `SET TRANSACTION` 语句之间尽量避免使用 TAB 补全，因为这可能导致后端执行 SQL 查询。

[2] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第十九集 [Transaction Isolation Levels](https://www.pgcasts.com/episodes/transaction-isolation-levels)。
