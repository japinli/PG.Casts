# 使用 citext 处理电子邮件

在这一集中，我们将研究如何在处理不区分大小写的数据时使用 citext 模块。在之前的剧集中，我们探讨了不区分大小写数据所带来的问题：

* [使用不区分大小写的数据](08-working-with-case-insensitive-data.md)
* [更多不区分大小写的数据](09-more-case-insensitive-data.md)

我们不得不处理一些障碍，以确保我们能够有效地查询我们的表，同时防止重复记录。我们学习了如何有效地查询我们的表并防止重复记录，但是有一种更简单的方法—— citext 模块。本文将介绍如何使用 citext。

首先，我们看看当前的 `users` 表：

```sql
\d old.users
```
```
                                     Table "old.users"
 Column |       Type        | Collation | Nullable |                Default
--------+-------------------+-----------+----------+---------------------------------------
 id     | integer           |           | not null | nextval('old.users_id_seq'::regclass)
 email  | character varying |           | not null |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
    "users_email_key" UNIQUE CONSTRAINT, btree (email)
    "users_unique_lower_email_idx" UNIQUE, btree (lower(email::text))
```

在此表中，我们的 `email` 列使用了 `varchar` 数据类型。在我们的新表中，我们将用 citext 模块提供的 `citext` 数据类型替换它。此数据类型的工作方式与 `text` 数据类型类似，但在比较期间，它在内部调用了 `lower()` 函数。

就像我说的，citext 将使事情变得更容易。在开始之前，我们需要先创建 citext 扩展。

```sql
CREATE EXTENSION citext;
```

然后，我们创建一个新的 `users` 表。

```sql
CREATE TABLE users (
  id serial primary key,
  email citext not null unique
);
```

我们可以快速浏览一下该表的结构：

```sql
\d users
```
```
                            Table "public.users"
 Column |  Type   | Collation | Nullable |              Default
--------+---------+-----------+----------+-----------------------------------
 id     | integer |           | not null | nextval('users_id_seq'::regclass)
 email  | citext  |           | not null |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
    "users_email_key" UNIQUE CONSTRAINT, btree (email)
```

让我们插入一个初始记录，以便我们可以使用一些东西，注意这里的大小写：

```sql
INSERT INTO users (email) VALUES ('PERSON1@example.com');
```

现在我们可以通过查询具有不同大小写的记录来查看不区分大小写的比较：

```sql
SELECT * FROM users WHERE email = 'person1@example.com';
```
```
 id |        email
----+---------------------
  1 | PERSON1@example.com
(1 row)
```

我们找到了记录，并且保留了原始插入的大小写。如果我们尝试插入具有不同大小写的重复记录将会怎样呢？

```sql
INSERT INTO users (email) VALUES ('Person1@example.com');
```
```
ERROR:  duplicate key value violates unique constraint "users_email_key"
DETAIL:  Key (email)=(Person1@example.com) already exists.
```

正如预期的那样，我们得到一个错误，我们的唯一性约束阻止了插入。最后，让我们看看性能比较。

我们可以在我们的新 `users` 表中插入一些额外的记录：

```sql
INSERT INTO users (email)
  SELECT 'person' || num || '@example.com'
  FROM generate_series(2, 10000) AS num;
```

现在，我们可以通过 `EXPLAIN ANALYZE` 来查看执行计划的输出：

```sql
EXPLAIN ANALYZE
  SELECT * FROM old.users
  WHERE lower(email) = lower('person5000@example.com');
```
```
                                                             QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using users_unique_lower_email_idx on users  (cost=0.29..8.30 rows=1 width=26) (actual time=0.042..0.050 rows=1 loops=1)
   Index Cond: (lower((email)::text) = 'person5000@example.com'::text)
 Planning Time: 0.128 ms
 Execution Time: 0.092 ms
(4 rows)
```

```sql
EXPLAIN ANALYZE
  SELECT * FROM users
  WHERE email = 'person5000@example.com';
```
```
                                                       QUERY PLAN
------------------------------------------------------------------------------------------------------------------------
 Index Scan using users_email_key on users  (cost=0.29..8.30 rows=1 width=26) (actual time=0.043..0.048 rows=1 loops=1)
   Index Cond: (email = 'person5000@example.com'::citext)
 Planning Time: 0.301 ms
 Execution Time: 0.085 ms
(4 rows)
```

我们可以看到，当使用具有唯一索引的 `citext` 字段时，我们获得了相当的性能。您可能想知道 `ilike` 运算符在哪里适合所有这些。我们将在下一集中看到这一点。在那之前，您的数据可能是一致的并且您的查询是高性能的。

## 拓展

PostgreSQL 是不推荐使用 citext，官方文档推荐使用[非确定性排序规则](https://www.postgresql.org/docs/current/collation.html#COLLATION-NONDETERMINISTIC)。在 PostgreSQL 中，排序规则（collation）包含确定性（deterministic）和非确定性（nondeterministic）两类。确定性排序规则使用确定性比较，这意味着只有当字符串由相同的字节序列组成时，它才认为它们是相等的。非确定性排序规则即使它们有不同的字节组成、也可以认为它们是相等的。

要创建非确定性排序规则，我们需要在 `CREATE COLLATION` 是指定 `deterministic = false`，例如：

```sql
CREATE COLLATION ndcoll (provider = icu, locale = 'und', deterministic = false);
```

那么我们尝试使用非确定性排序规则来实现上文的大小写需求。首先我需要创建一个大小写不敏感的排序规则。

```sql
CREATE COLLATION case_insensitive (provider = icu, locale = 'und-u-ks-level2', deterministic = false);
```

接着，我们在 `new` 模式下创建一个 `users` 表。

```sql
CREATE SCHEMA new;
CREATE TABLE new.users (
  id serial primary key,
  email varchar not null unique collate case_insensitive
);
```

`users` 表的结构信息如下：

```sql
\d new.users
```
```
                                        Table "new.users"
 Column |       Type        |    Collation     | Nullable |                Default
--------+-------------------+------------------+----------+---------------------------------------
 id     | integer           |                  | not null | nextval('new.users_id_seq'::regclass)
 email  | character varying | case_insensitive | not null |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
    "users_email_key" UNIQUE CONSTRAINT, btree (email)
```

同样地，我们插入一个初始记录（注意大小写）：

```sql
INSERT INTO new.users (email) VALUES ('PERSON1@example.com');
```

接着以不同的大小写查询记录：

```sql
SELECT * FROM new.users WHERE email = 'person1@example.com';
```
```
 id |        email
----+---------------------
  1 | PERSON1@example.com
(1 row)
```

这个的结果和上面使用 citext 完全相同。那么插入具有不同大小写的重复记录将会怎样呢？

```sql
INSERT INTO new.users (email) VALUES ('Person1@example.com');
```
```
ERROR:  duplicate key value violates unique constraint "users_email_key"
DETAIL:  Key (email)=(Person1@example.com) already exists.
```

我们的唯一性约束也得到了保证。我们在 `new.users` 表中在插入一些额外的记录：

```sql
INSERT INTO new.users (email)
  SELECT 'person' || num || '@example.com'
  FROM generate_series(2, 10000) AS num;
```

通过 `EXPLAIN ANALYZE` 来查看执行计划的输出：

```sql
EXPLAIN ANALYZE
  SELECT * FROM new.users
  WHERE email = 'person5000@example.com';
```
```
                                                       QUERY PLAN
------------------------------------------------------------------------------------------------------------------------
 Index Scan using users_email_key on users  (cost=0.29..8.30 rows=1 width=26) (actual time=0.039..0.049 rows=1 loops=1)
   Index Cond: ((email)::text = 'person5000@example.com'::text)
 Planning Time: 0.507 ms
 Execution Time: 0.107 ms
(4 rows)
```

## 演示示例

```sql
CREATE SCHEMA old;
CREATE TABLE old.users (
  id serial primary key,
  email varchar not null unique
);
CREATE UNIQUE INDEX users_unique_lower_email_idx ON old.users (lower(email));

INSERT INTO old.users (email)
  SELECT 'person' || num || '@example.com'
  FROM generate_series(1,10000) AS num;
```

## 参考

[1] https://www.postgresql.org/docs/current/citext.html

[2] https://www.postgresql.org/docs/current/collation.html#COLLATION-NONDETERMINISTIC

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第十三集 [Citext for Emails](https://www.pgcasts.com/episodes/citext-for-emails)。

[2] 原文中的 `Setup` 部分包含重复的 SQL 语句，翻译时将其删除了。
