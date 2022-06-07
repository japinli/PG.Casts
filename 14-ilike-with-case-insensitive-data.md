# ILIKE 与不区分大小写的数据

在这一集中，我们将研究 `ILIKE` 运算符，看看为什么它可能不适合进行完全不区分大小写的字符串比较。

在前几集中，我们研究了处理不区分大小写数据的不同方法。我们希望在不牺牲性能的情况下以不区分大小写的方式通过电子邮件地址查询用户记录。一种方法是结合函数索引使用 `lower()` 函数进行比较。另一种方法是引入 citext 模块。

这两种方法都能完成工作，并且具有非常相似的性能特征。但是，它们要求我们通过创建额外的索引或引入扩展来跳过一些障碍。您可能想知道，仅使用 `ilike` 运算符进行不区分大小写的字符串比较不是更容易吗？

让我们来看看。 我们已经有一个包含 10,000 条记录的 `users` 表。

```sql
\d users
```
```
                                 Table "public.users"
 Column |       Type        | Collation | Nullable |              Default
--------+-------------------+-----------+----------+-----------------------------------
 id     | integer           |           | not null | nextval('users_id_seq'::regclass)
 email  | character varying |           | not null |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
    "users_unique_lower_email_idx" UNIQUE, btree (lower(email::text))
```

```sql
SELECT * FROM users LIMIT 10;
```
```
 id |        email
----+----------------------
  1 | person1@example.com
  2 | person2@example.com
  3 | person3@example.com
  4 | person4@example.com
  5 | person5@example.com
  6 | person6@example.com
  7 | person7@example.com
  8 | person8@example.com
  9 | person9@example.com
 10 | person10@example.com
(10 rows)
```

首先，让我们在查询用户记录的时候执行 `EXPLAIN ANALYZE`。

```sql
EXPLAIN ANALYZE
  SELECT * FROM users WHERE lower(email) = lower('person5000@example.com');
```
```
                                                             QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using users_unique_lower_email_idx on users  (cost=0.29..8.30 rows=1 width=26) (actual time=0.033..0.040 rows=1 loops=1)
   Index Cond: (lower((email)::text) = 'person5000@example.com'::text)
 Planning Time: 0.332 ms
 Execution Time: 0.088 ms
(4 rows)
```

现在，让我们做一个类似的查询，不使用 `lower()` 函数，而是使用 `ILIKE` 运算符。

```sql
EXPLAIN ANALYZE
  SELECT * FROM users WHERE email ILIKE 'person5000@example.com';
```
```
                                             QUERY PLAN
----------------------------------------------------------------------------------------------------
 Seq Scan on users  (cost=0.00..199.00 rows=1 width=26) (actual time=11.140..18.624 rows=1 loops=1)
   Filter: ((email)::text ~~* 'person5000@example.com'::text)
   Rows Removed by Filter: 9999
 Planning Time: 0.251 ms
 Execution Time: 18.712 ms
(5 rows)
```

`ILIKE` 查询的执行速度要慢得多。它强制 PostgreSQL 进行完整的顺序扫描。这是因为它无法利用我们的函数 b-tree 索引。它也无法使用更基本的 b-tree 索引。`ILIKE` 运算符及类似的运算符旨在进行模式匹配比较，因此它们无法利用 b-tree 索引。

对于这些完全不区分大小写的字符串比较，我们最好使用前面讨论的方法。也就是说，您会认为我们在使用模式匹配运算符时能够进行索引查询。trigram 模块和 gist 索引可以帮助我们，但这是另一集的主题。在那之前，您的数据可能是一致的并且您的查询是高性能的。

## 演示示例

```sql
CREATE TABLE users (
  id serial primary key,
  email varchar not null
);
CREATE UNIQUE INDEX users_unique_lower_email_idx ON users (lower(email));

INSERT INTO users (email)
  SELECT 'person' || num || '@example.com'
  FROM generate_series(1, 10000) AS num;
```

## 参考

[1] [PostgreSQL's Pattern Matching Operators](http://www.postgresql.org/docs/current/static/functions-matching.html)

[2] [Citext for emails](https://www.pgcasts.com/episodes/13/citext-for-emails)

[3] [Working With Case-Insensitive Data](https://www.pgcasts.com/episodes/8/working-with-case-insensitive-data) and [More Case-Insensitive Data](https://www.pgcasts.com/episodes/9/more-case-insensitive-data)

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第十四集 [ILIKE with Case Insensitive Data](https://www.pgcasts.com/episodes/ilike-with-case-insensitive-data)。
