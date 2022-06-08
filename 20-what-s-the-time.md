# 时间戳

在本集中 Josh Branchaud 将介绍 PostgreSQL 提供的 3 种不同类型的时间戳函数。说到时间，PostgreSQL 可以做很多事情。即使您只是想知道现在的时间，PostgreSQL 中也有几个不同的函数需要考虑。

我们最熟悉的可能是 `now()`：

```sql
SELECT now();
```
```
              now
-------------------------------
 2022-05-16 21:07:57.435514+08
(1 row)
```

该函数给了我们当前时间。`now()` 函数的工作方式与 `transaction_timestamp()` 函数相同：

```sql
SELECT transaction_timestamp();
```
```
     transaction_timestamp
-------------------------------
 2022-05-16 21:09:10.761453+08
(1 row)
```

我们将在本集中继续使用 `transaction_timestamp()`，但请记住它可以与 `now()` 互换。我们将要研究的另外两个时间戳函数是 `statement_timestamp()` 和 `clock_timestamp()`：

```sql
SELECT statement_timestamp();
```
```
      statement_timestamp
-------------------------------
 2022-05-16 22:16:46.474594+08
(1 row)
```
```sql
SELECT clock_timestamp();
```
```
        clock_timestamp
-------------------------------
 2022-05-16 22:17:13.996596+08
(1 row)
```

它们似乎都在做同样的事情，那么，它们有什么区别呢？它们的名字可能会让我们了解其的真正作用。让我们开始一个事务，仔细看看：

```sql
BEGIN;

SELECT transaction_timestamp();
```
```
     transaction_timestamp
-------------------------------
 2022-05-16 22:19:12.813916+08
(1 row)
```

我们可以等待几秒钟，然后再次尝试该语句：

```sql
SELECT transaction_timestamp();
```
```
     transaction_timestamp
-------------------------------
 2022-05-16 22:19:12.813916+08
(1 row)
```

有趣的是，它们都产生完全相同的时间戳。那是因为这个函数返回当前事务开始的时间戳。现在，让我们试试 `statement_timestamp()` 函数：

```sql
SELECT statement_timestamp();
```
```
     statement_timestamp
------------------------------
 2022-05-16 22:20:53.62437+08
(1 row)
```

自我们开始事务以来已经过去了一段时间，这在此处有所体现。`statement_timestamp()` 函数提供 PostgreSQL 开始执行语句的时间戳。

因此，即使我们有一个昂贵的查询，多次引用 `statement_timestamp()`，我们每次都会得到相同的时间戳：

```sql
SELECT statement_timestamp(), pg_sleep(2)::text
UNION ALL
SELECT statement_timestamp(), '';
```
```
      statement_timestamp      | pg_sleep
-------------------------------+----------
 2022-05-16 22:23:16.326286+08 |
 2022-05-16 22:23:16.326286+08 |
(2 rows)
```

我们使用 `pg_sleep()` 函数来模拟一个昂贵的查询。尽管对 `statement_timestamp()` 的两次不同调用之间有几秒钟的时间，但它们都产生完全相同的时间戳。这就是 `clock_timestamp()` 出现的地方。让我们尝试与上面相同的查询，但使用 `clock_timestamp()` 代替 `statement_timestamp()`：

```sql
SELECT clock_timestamp(), pg_sleep(2)::text
UNION ALL
SELECT clock_timestamp(), '';
```
```
        clock_timestamp        | pg_sleep
-------------------------------+----------
 2022-05-16 22:24:49.802825+08 |
 2022-05-16 22:24:51.804835+08 |
(2 rows)
```

我们可以看到此示例的时间戳差异，因为 `clock_timestamp()` 函数获取实际的当前时间。您现在知道不同时间戳函数之间的区别。如果我们在事务中更新或创建许多不同的记录，并且我们希望协调它们的所有时间戳，我们应该使用 `transaction_timestamp()` 或者我们可以使用 `now()` 作为简写。当需要更具体的时间戳时，您可能需要使用 `statement_timestamp()` 或 `clock_timestamp()`。

感谢 Jack Christensen 的推荐，我使用 `pg_sleep(2)::text` 作为简化模拟昂贵查询的一种方式。

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第二十集 [What's the Time](https://www.pgcasts.com/episodes/what-s-the-time)。
