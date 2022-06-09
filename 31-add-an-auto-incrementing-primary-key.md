# 添加自增主键

大家好，今天我们来看看如何在 PostgreSQL 中为表添加自增主键。

在这个例子中，我们将处理一个具有表 `users` 的虚拟数据库，当前没有设置主键。我们可以使用 `\d` 命令检查这个表。

```sql
\d users
```
```
                    Table "public.users"
 Column |       Type        | Collation | Nullable | Default
--------+-------------------+-----------+----------+---------
 name   | character varying |           | not null |
 email  | character varying |           | not null |
 age    | integer           |           |          |
Indexes:
    "users_email_key" UNIQUE CONSTRAINT, btree (email)
```

我们也可以查询这个表，看看我们已经有的数据。

```sql
SELECT * FROM users;
```
```
 name |      email       | age
------+------------------+-----
 Sam  | sam@example.com  |  35
 Sue  | sue@example.com  |  65
 Carl | carl@example.com |  45
(3 rows)
```

要添加一个自动递增的 `id` 作为我们表的主键，我们可以使用 `ALTER TABLE` 命令，传入我们的 `users` 表，然后指定我们要添加一个名为 `id` 的列，类型为 `serial` 并设置它作为我们的主键。

```sql
ALTER TABLE users ADD COLUMN id serial primary key;
```

`serial` 类型是 PostgreSQL 的简写，它允许我们创建具有自动递增的整数列。现在，如果我们检查 `users` 表，我们可以看到有一个整数类型的新列，其中包含 PostgreSQL 为我们生成的序列。

```sql
\d users
```
```
                                 Table "public.users"
 Column |       Type        | Collation | Nullable |              Default
--------+-------------------+-----------+----------+-----------------------------------
 name   | character varying |           | not null |
 email  | character varying |           | not null |
 age    | integer           |           |          |
 id     | integer           |           | not null | nextval('users_id_seq'::regclass)
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
    "users_email_key" UNIQUE CONSTRAINT, btree (email)
```

如果我们再次查询 `users` 表，我们可以看到我们现有的用户已经填充了 `id`。

```sql
SELECT * FROM users;
```
```
 name |      email       | age | id
------+------------------+-----+----
 Sam  | sam@example.com  |  35 |  1
 Sue  | sue@example.com  |  65 |  2
 Carl | carl@example.com |  45 |  3
(3 rows)
```

最后，让我们在表中添加一个新用户。

```sql
INSERT INTO users VALUES ('Jamie', 'EM3456');
```

再次查询 `users` 表，我们可以看到新行已经适当地填充了 `id`。

```sql
SELECT * FROM users;
```
```
 name  |      email       | age | id
-------+------------------+-----+----
 Sam   | sam@example.com  |  35 |  1
 Sue   | sue@example.com  |  65 |  2
 Carl  | carl@example.com |  45 |  3
 Jamie | EM3456           |     |  4
(4 rows)
```

## 演示示例

```sql
CREATE TABLE users (
  name varchar not null,
  email varchar not null unique,
  age integer
);

INSERT INTO users VALUES
  ('Sam', 'sam@example.com', 35),
  ('Sue', 'sue@example.com', 65),
  ('Carl', 'carl@example.com', 45);
```

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第三十一集 [Add an Auto Incrementing Primary Key](https://www.pgcasts.com/episodes/add-an-auto-incrementing-primary-key)。
