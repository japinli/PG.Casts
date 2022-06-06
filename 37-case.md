# Case

大家好，今天我们来看看如何在 PostgreSQL 中使用 `case` 语句。

PostgreSQL 中的 `case` 语句是一种条件语句，类似于其他编程语言中的 `if/else` 语句。`Case` 语句遵循 `when/then/else` 流程。让我们看看这是如何工作的。

在此示例中，我有一个名为 `users` 的测试表，其中包含一些记录。

```sql
\d users
```
```
                                    Table "public.users"
   Column    |       Type        | Collation | Nullable |              Default
-------------+-------------------+-----------+----------+-----------------------------------
 id          | integer           |           | not null | nextval('users_id_seq'::regclass)
 name        | character varying |           | not null |
 posts_count | integer           |           |          |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
```

```sql
SELECT * FROM users;
```
```
 id |  name   | posts_count
----+---------+-------------
  1 | Abe     |           0
  2 | Boris   |          16
  3 | Cameron |           3
  4 | Dave    |           7
  5 | Eve     |          54
  6 | Frank   |           Ø
(6 rows)
```

您会注意到我们的用户有一个帖子计数。如果我们想从我们的表中读取数据，并根据用户发布的帖子数量返回类似贡献评级的内容，我们可以使用 `case` 语句。

让我们试试这个。 我将使用 `\e` 命令来使用 VIM 作为我的查询编辑器。

```sql
\e
```

我们将从表中选择 `name` 和 `posts_count` 开始，然后我们将添加到我们的 `case` 语句中以确定贡献评级。

基本的 `case` 语句可以接受一个表达式，并根据 `when` 子句中的条件评估该表达式。因此，如果我们将 `posts_count` 传递给我们的 `case` 语句，并在我们的 `when` 子句中指定一个数字，我们正在检查我们的帖子计数和我们在 `when` 子句中的值之间的完全匹配。

```sql
SELECT
  name,
  posts_count,
  case
    posts_count when 0 then 'Novice'
  end as contribution_rating
FROM
  users;
```
```
  name   | posts_count | contribution_rating
---------+-------------+---------------------
 Abe     |           0 | Novice
 Boris   |          16 |
 Cameron |           3 |
 Dave    |           7 |
 Eve     |          54 |
 Frank   |             |
(6 rows)
```

正如我们在此处的输出中看到的那样，只有一个用户满足了 `case` 语句中的 `when` 子句条件。我们其他用户的贡献评级为空值。

在这里，我们只是要返回 `Unknown`。

```sql
SELECT
  name,
  posts_count,
  case posts_count
    when 0 then 'Novice'
    else 'Unknown'
  end as contribution_rating
FROM
  users;
```
```
  name   | posts_count | contribution_rating
---------+-------------+---------------------
 Abe     |           0 | Novice
 Boris   |          16 | Unknown
 Cameron |           3 | Unknown
 Dave    |           7 | Unknown
 Eve     |          54 | Unknown
 Frank   |             | Unknown
(6 rows)
```

现在，我们可以看到我们所有的用户都有一个贡献评级，但除了一个之外，所有用户都是 `Unknown`。显然，我们需要一种不同的方法，因此我们不必直接匹配帖子计数中的所有可能结果。

为此，我们将稍作改动。我们可以将所有逻辑直接移动到 `when` 子句中，而不是为我们的 `case` 语句提供一个表达式来评估我们的 `when` 子句。
在我们的 `when` 子句中，我们可以包含子表达式。对我们来说，这意味着我们可以使用不等式来为我们的贡献评级获得更有意义的输出。

因此，如果用户有超过 30 个帖子，他们的评级是 `Gold`。如果他们有超过 10 个帖子，他们的评级是 `Silver`。您会注意到，对于我们的第二个 `when` 子句，我没有指定 `posts_count` 也必须小于我们第一个 `when` 子句的值。这是因为 PostgreSQL 中的 `case` 语句将在第一个匹配的 `when` 子句上返回；不会评估所有后续的 `when` 子句。

```sql
SELECT
  name,
  posts_count,
  case
    when posts_count > 30 then 'Gold'
    when posts_count > 10 then 'Silver'
    when posts_count > 0 then 'Bronze'
    when posts_count = 0 then 'Novice'
    else 'Unknown'
  end as contribution_rating
FROM
  users;
```
```
  name   | posts_count | contribution_rating
---------+-------------+---------------------
 Abe     |           0 | Novice
 Boris   |          16 | Silver
 Cameron |           3 | Bronze
 Dave    |           7 | Bronze
 Eve     |          54 | Gold
 Frank   |             | Unknown
(6 rows)
```

从现在的输出中，我们可以看到除了我们的一个用户之外的所有用户都有一个已知的贡献评级。我们的最后一个 `Unknown` 用户的 `posts_count` 为空值。那么如果我们希望没有 `posts_count` 数据的用户也被过滤到 `Novice` 贡献评级中该怎么做呢？

我们可以通过扩展最后一个 `when` 子句的子表达式来做到这一点，添加一个 `or` 并检查 `posts_count` 为空。

```sql
SELECT
  name,
  posts_count,
  case
    when posts_count > 30 then 'Gold'
    when posts_count > 10 then 'Silver'
    when posts_count > 0 then 'Bronze'
    when posts_count = 0 or posts_count is null then 'Novice'
    else 'Unknown'
  end as contribution_rating
FROM
  users;
```
```
  name   | posts_count | contribution_rating
---------+-------------+---------------------
 Abe     |           0 | Novice
 Boris   |          16 | Silver
 Cameron |           3 | Bronze
 Dave    |           7 | Bronze
 Eve     |          54 | Gold
 Frank   |             | Novice
(6 rows)
```

我们现在可以看到，我们的 `posts_count` 为空值的用户已获得 `Novice` 贡献评级。

## 演示示例

```sql
CREATE TABLE users (
  id serial primary key,
  name varchar not null,
  posts_count integer
);

INSERT INTO users (name, posts_count) VALUES
  ('Abe', 0), ('Boris', 16), ('Cameron', 3),
  ('Dave', 7), ('Eve', 54), ('Frank', null);
```

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第三十七集 [Case](https://www.pgcasts.com/episodes/case)
