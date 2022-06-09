# Upserts

大家好，今天我们来看看如何在 PostgreSQL 中进行 `upserts` 操作。

`upsert` 是指在满足某些条件的情况下，插入新记录或者更新数据库中已有记录的语句。在 PostgreSQL 中，可以在插入期间使用 `ON CONFLICT` 子句来完成 `upsert` 语句。

`ON CONFLICT` 子句允许我们在发生唯一冲突或排除约束冲突时指定与 `INSERT` 不同的行为。`ON CONFLICT` 子句允许我们指定两种不同的操作：什么都不做，或者做更新。

让我们看看这是如何工作的。对于此示例，我创建了一个带有名为 `users` 表的虚拟数据库。`users` 表有几列，其中一列用于电子邮件，并且具有唯一性且非空约束。

```sql
\d users
```
```
                    Table "public.users"
 Column |       Type        | Collation | Nullable | Default
--------+-------------------+-----------+----------+---------
 name   | character varying |           | not null |
 email  | text              |           | not null |
 age    | integer           |           |          |
Indexes:
    "users_email_key" UNIQUE CONSTRAINT, btree (email)
```

## ON CONFLICT DO NOTHING

我将使用 `\e` 命令使用 vim 用作我的查询缓冲区编辑器。默认情况下，如果没有为 `INSERT` 语句提供 `ON CONFLICT` 子句，则会因违反约束而引发错误。我们可以通过尝试使用我们数据库中已经存在的电子邮件创建一个新用户来看到这一点。

```sql
INSERT INTO users VALUES ('Sam Smith', 'sam@example.com', 38);
```
```
ERROR:  duplicate key value violates unique constraint "users_email_key"
DETAIL:  Key (email)=(sam@example.com) already exists.
```

在这里我们可以看到 PostgreSQL 返回了一个违反唯一性约束的错误。如果我们想忽略此类错误，我们可以使用 `ON CONFLICT` 子句并指定 `DO NOTHING`。

```sql
INSERT INTO users VALUES ('Sam Smith', 'sam@example.com', 38)
  ON CONFLICT do nothing;
```

我们可以从输出中看到我们的命令没有进行任何更改，但也没有引发错误。

## ON CONFLICT DO UPDATE

如果我们想在发生冲突时执行更新操作，我们需要指定 `DO UPDATE`。`DO UPDATE` 语句需要参数来匹配：冲突的列，或者被违反的约束。要在列上进行匹配，我们只需在 `ON CONFLICT` 语句之后指定列名。在我们的例子中，我们的唯一约束设置在 `email` 上，所以我们要做的是 `ON CONFLICT (email)`。

在我们告诉 PostgreSQL 我们希望在发生冲突时执行更新操作后，我们必须指定如何执行更新。这是使用 `set` 命令完成的，并提供我们希望更新的列名和值。

在 PostgreSQL 中，有一个名为 `excluded` 的特殊表，用于表示建议插入的行。正是从这个表中，我们能够获得更新语句的值。

```sql
INSERT INTO users VALUES ('Sam Smith', 'sam@example.com', 38)
  ON CONFLICT (email) do update
  SET name = excluded.name, age = excluded.age;
```

现在我们可以从输出中看到我们已经进行了更改。

```sql
TABLE users;
```
```
   name    |      email       | age
-----------+------------------+-----
 Sue       | sue@example.com  |  24
 Carl      | carl@example.com |  26
 Sam Smith | sam@example.com  |  38
(3 rows)
```

## ON CONFLICT ON CONSTRAINT

如果在 `ON CONFLICT` 期间我们想要匹配约束而不是列，我们需要使用 `ON CONSTRAINT`，然后是我们要匹配的约束。

```sql
INSERT INTO users VALUES ('Sam Smith', 'sam@example.com', 40)
  ON CONFLICT ON CONSTRAINT users_email_key do update
  SET name = excluded.name, age = excluded.age;
```

我们同样可以看到我们的数据库已经更新。

```sql
TABLE users;
```
```
   name    |      email       | age
-----------+------------------+-----
 Sue       | sue@example.com  |  24
 Carl      | carl@example.com |  26
 Sam Smith | sam@example.com  |  40
(3 rows)
```

## 演示示例

```sql
CREATE TABLE users (
  name varchar not null,
  email text not null unique,
  age integer
);

INSERT INTO users VALUES
  ('Sam', 'sam@example.com', 35),
  ('Sue', 'sue@example.com', 24),
  ('Carl', 'carl@example.com', 26);
```


## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第三十集 [Upserts](https://www.pgcasts.com/episodes/upserts)。

[2] 在 PostgreSQL 15 版本中引入了 [`MERGE`](https://www.postgresql.org/docs/15/sql-merge.html) 语法也可以实现类似的功能。
