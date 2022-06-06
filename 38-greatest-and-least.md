# Greatest 和 Least

大家好，今天我们来看看 PostgreSQL 中的 `greatest` 和 `least` 语句。

`Greatest` 和 `least` 是 PostgreSQL 中的条件语句，它们接受参数列表，从参数中返回最大值和最小值。
让我们看看这是如何工作的。

在此示例中，我有一个名为 `users` 的表，其中包含一些记录。

```sql
\d users
```
```
                                       Table "public.users"
      Column       |       Type        | Collation | Nullable |              Default
-------------------+-------------------+-----------+----------+-----------------------------------
 id                | integer           |           | not null | nextval('users_id_seq'::regclass)
 name              | character varying |           | not null |
 last_post_date    | date              |           |          |
 last_comment_date | date              |           |          |
 sign_up_date      | date              |           |          |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
```

```sql
SELECT * FROM users;
```
```
 id | name | last_post_date | last_comment_date | sign_up_date
----+------+----------------+-------------------+--------------
  1 | Amy  | 2019-10-05     | 2019-11-04        | 2019-09-10
  2 | Brad | 2019-10-20     | 2019-08-11        | 2019-07-23
  3 | Chad | 2019-11-03     | 2019-11-02        | 2019-10-31
  4 | Dana | 2019-09-29     |                   | 2019-08-04
(4 rows)
```

通过查询 `users` 表，您会注意到我们的用户有几个日期列，包括注册日期、最后发布日期和最后评论日期。

如果我们想从 `users` 表中读取并找到类似上次活动日期的内容，我们可以使用 `greatest`，将注册日期、最后发布日期和最后评论日期作为参数传递给 `greatest`。

```sql
SELECT
  name,
  greatest(sign_up_date, last_post_date, last_comment_date) as last_activity_date
from
  users;
```
```
 name | last_activity_date
------+--------------------
 Amy  | 2019-11-04
 Brad | 2019-10-20
 Chad | 2019-11-03
 Dana | 2019-09-29
(4 rows)
```

你会注意到我们所有的用户都有一个最后的活动日期，即使我们最后一个名为 `Dana` 的用户没有最后评论日期。这是因为 `greatest` 和 `least` 语句会忽略其参数列表中的空值。只有当参数列表中的所有值都为空时，这些方法才会返回空。

如果我们创建一个新用户，然后执行之前的相同 `SELECT` 语句，我们可以看到这一点。

```sql
INSERT INTO users (name) VALUES ('Ed');
```

```sql
SELECT
  name,
  greatest(sign_up_date, last_post_date, last_comment_date) as last_activity_date
from
  users;
```
```
 name | last_activity_date
------+--------------------
 Amy  | 2019-11-04
 Brad | 2019-10-20
 Chad | 2019-11-03
 Dana | 2019-09-29
 Ed   |
(5 rows)
```

`Least` 命令的工作方式与 `greatest` 完全相同，接受一个值列表并返回最小的值。对于我们的数据，我们可以做一些愚蠢的事情，比如使用 `least` 从注册日期、最后评论日期和最后发布日志中找到用户最早的活动日期<sup>[1]</sup>。

```sql
SELECT
  name,
  least(sign_up_date, last_comment_date, last_post_date) as earliest_activity_date
from
  users;
```
```
 name | earliest_activity_date
------+------------------------
 Amy  | 2019-09-10
 Brad | 2019-07-23
 Chad | 2019-10-31
 Dana | 2019-08-04
 Ed   |
(5 rows)
```

需要注意的是，`greatest` 和 `least` 命令与 PostgreSQL 中的 `max` 和 `min` 聚合函数之间的巨大差异。`Greatest` 和 `least` 为正在读取的每一行计算不同的值，它们接受一个参数列表。`Max` 和 `min` 基于正在读取的行中的所有值计算单个结果，并且只接受一个参数。

因此，使用 `max` 和最后一次发布日期作为我们用户的最新帖子会读取我们每个用户行的最后一次发布日期，并返回我们所有用户的所有最后一次发布日期中最大的一个。

```sql
SELECT max(last_post_date) as latest_post FROM users;
```
```
 latest_post
-------------
 2019-11-03
(1 row)
```

这与使用最晚发布日期和最后评论日期来查找我们用户的最新帖子非常不同。`Greatest` 返回每个用户的最新帖子。

```sql
SELECT
  name,
  greatest(last_post_date, last_comment_date) as latest_post
FROM
  users;
```
```
 name | latest_post
------+-------------
 Amy  | 2019-11-04
 Brad | 2019-10-20
 Chad | 2019-11-03
 Dana | 2019-09-29
 Ed   |
(5 rows)
```

## 演示示例

```sql
CREATE TABLE users (
  id serial primary key,
  name varchar not null,
  last_post_date date,
  last_comment_date date,
  sign_up_date date
);

INSERT INTO users (name, last_post_date, last_comment_date, sign_up_date) VALUES
  ('Amy', '2019-10-05', '2019-11-04', '2019-09-10'),
  ('Brad', '2019-10-20', '2019-08-11', '2019-07-23'),
  ('Chad', '2019-11-03', '2019-11-02', '2019-10-31'),
  ('Dana', '2019-09-29', null, '2019-08-04');
```

## 译者著

[1] 原文中这里没有选择 `name` 字段。

[2] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第三十八集 [Greatest and Least](https://www.pgcasts.com/episodes/greatest-and-least)。
