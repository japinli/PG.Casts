# Coalesce

大家好，今天我们将探索 PostgreSQL 中的 `coalesce` 条件语句。

PostgreSQL 中的 `coalesce` 函数接受一个参数列表，并将返回列表中第一个不为空的参数。让我们试试这个。在此示例中，我有一个名为 `posts` 的测试表，其中包含一些记录。

```sql
\d posts
```
```
                                       Table "public.posts"
      Column       |       Type        | Collation | Nullable |              Default
-------------------+-------------------+-----------+----------+-----------------------------------
 id                | integer           |           | not null | nextval('posts_id_seq'::regclass)
 title             | character varying |           | not null |
 description       | text              |           |          |
 short_description | character varying |           |          |
 content           | text              |           |          |
 author            | character varying |           |          |
 release_date      | date              |           |          |
Indexes:
    "posts_pkey" PRIMARY KEY, btree (id)
```

正如您在表结构中看到的，我们有两列是和描述相关的。如果存在 `short_description`，我们就读取它并返回，否则返回 `description`。我们将使用 `coalesce` 语句，首先将 `short_description` 作为其参数，然后是 `description`。

```sql
SELECT coalesce(short_description, description) AS display_description FROM posts;
```
```
                display_description
---------------------------------------------------
 In this episode we explore the COALESCE statement
 This description is already short
(2 rows)
```

如果提供给 `coalesce` 的所有值都为 `null`，则该语句将返回 `null`。当我们尝试在 `author` 上使用 `coalesce` 时，我们便可以看到这一点。

```sql
SELECT coalesce(author) FROM posts;
```
```
 coalesce
----------
 Mary Lee

(2 rows)
```

在这种情况下，如果 `author` 不存在，我们还可以使用 `coalesce` 为 `author` 提供默认值。

```sql
SELECT coalesce(author, 'Unknown') FROM posts;
```
```
 coalesce
----------
 Mary Lee
 Unknown
(2 rows)
```

关于 `coalesce` 的一个小警告是，您不能在参数列表中混用数据类型。如果我们使用 `release_date` 并尝试在日期为空时给出 `Draft` 的默认值，我们就可以看到这一点。

```sql
SELECT coalesce(release_date, 'Draft') FROM posts;
```
```
ERROR:  invalid input syntax for type date: "Draft"
LINE 1: SELECT coalesce(release_date, 'Draft') FROM posts;
                                      ^
```

如您所见，PostgreSQL 抛出错误，因为我们的字符串 `Draft` 不是有效日期。

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
    'How to Coalesce',
    'In this episode of PG Casts, we are going to explore the COALESCE conditional statement in Postgres',
    'In this episode we explore the COALESCE statement',
    null,
    'Mary Lee',
    '2019-01-31'
  ), (
    'This is a post',
    'This description is already short',
    null,
    'Content goes here',
    null,
    null
  );
```

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第三十五集 [Coalesce](https://www.pgcasts.com/episodes/coalesce)。
