# Nullif

大家好，今天我们将探索 PostgreSQL 中的 `nullif` 语句。

PostgreSQL 中的 `nullif` 语句接受两个参数，如果两个参数相等则返回 `null`，否则返回第一个参数。

让我们看看它是如何工作的。对于此示例，我有一个名为 `posts` 的表，其中包含一些记录。

```sql
SELECT * FROM posts;
```
```
 id |       title       |                                            description                                            |                short_description                |      content      |  author  | release_date
----+-------------------+---------------------------------------------------------------------------------------------------+-------------------------------------------------+-------------------+----------+--------------
  1 | How to Use Nullif | In this episode of PG Casts, we are going to explore the NULLIF conditional statement in Postgres | In this episode we explore the NULLIF statement |                   | Mary Lee | 2019-01-31
  2 | This is a post    | This description is already short                                                                 |                                                 | Content goes here |          |
(2 rows)
```

如果我们想在查询时删除 `short_description` 列中的空白字符，我们可以使用 `nullif` 语句，将 `short_description` 作为其第一个参数，然后是一个空字符串作为其第二个参数<sup>[1]</sup>。

```sql
SELECT nullif(short_description, '') AS short_description FROM posts;
```
```
                short_description
-------------------------------------------------
 In this episode we explore the NULLIF statement
 Ø
(2 rows)
```

从输出中可以看出，我们的 `short_description` 不为空白的行返回了 `short_description`。但是，当 `short_description` 为空白行时，它返回了 `null` 而不是空字符串。

当与 `coalesce` 语句结合使用时，`nullif` 语句会非常有用。例如，如果我们想从我们的表中读取并返回 `short_description`（如果存在），否则返回 `description`：

```sql
SELECT coalesce(short_description, description) AS display_description FROM posts;
```
```
               display_description
-------------------------------------------------
 In this episode we explore the NULLIF statement

(2 rows)
```

我们可以看到，在 `coalesce` 语句中，空字符串 `short_description` 不会被视为为 `null`，因此我们的输出中有空白值。

为了解决这个问题，我们可以将 `nullif` 语句与 `coalesce` 语句一起使用，将其用作 `coalesce` 的第一个参数，并将我们的 `short_description` 传递给它，然后是一个空字符串。

```sql
SELECT
  coalesce(nullif(short_description, ''), description) AS display_description
from
  posts;
```
```
               display_description
-------------------------------------------------
 In this episode we explore the NULLIF statement
 This description is already short
(2 rows)
```

从输出中，我们现在可以看到我们的两行都有一个描述信息。

## 演示示例

```sql
CREATE TABLE posts (
  id serial primary key,
  title varchar not null,
  description text,
  short_description varchar,
  content text,
  author varchar,
  release_date date
);

INSERT INTO posts (title, description, short_description, content, author, release_date) VALUES
  (
    'How to Use Nullif',
    'In this episode of PG Casts, we are going to explore the NULLIF conditional statement in Postgres',
    'In this episode we explore the NULLIF statement',
    null,
    'Mary Lee',
    '2019-01-31'
  ), (
    'This is a post',
    'This description is already short',
    '',
    'Content goes here',
    null,
    null
  );
```

## 译者著

[1] 为了区分 `null` 和空字符串，我使用了 `\pset null 'Ø'` 设置 `null` 的显示。

[2] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第三十六集 [Nullif](https://www.pgcasts.com/episodes/nullif)。
