# 添加和删除列

在这一集中，我们将探索如何在 PostgreSQL 的表中添加和删除列。

在此示例中，我们将处理一个虚拟数据库，该数据库具有一个名为 `users` 的表，其中包含列 `name` 和 `employee number`。

```sql
SELECT * FROM users ;
```
```
 name  | employee_number
-------+-----------------
 Sam   | EM1234
 Emily | EM2345
(2 rows)
```

## ADD COLUMN

要向我们的表中添加一个新列，我们需要使用 `ALTER TABLE` 命令来更改我们的表。然后我们指定要对表执行的操作，在我们的例子中是 `ADD COLUMN`。最后，我们指定我们想要的列名和类型，以及我们可能想要的任何约束。在这里，我添加了一个名为 `email` 的新列，它具有唯一性约束。

```sql
ALTER TABLE users ADD COLUMN email varchar unique;
```

现在我们可以检查我们的表并查看我们的新列和约束。

```sql
\d users
```
```
                         Table "public.users"
     Column      |       Type        | Collation | Nullable | Default
-----------------+-------------------+-----------+----------+---------
 name            | character varying |           | not null |
 employee_number | character varying |           | not null |
 email           | character varying |           |          |
Indexes:
    "users_email_key" UNIQUE CONSTRAINT, btree (email)
    "users_employee_number_key" UNIQUE CONSTRAINT, btree (employee_number)
```

## DROP COLUMN

要从我们的表中删除一列，我们使用与上面相同的 `ALTER TABLE` 命令，再次将 `users` 作为我们的表名。然而，这一次，我们的操作将是 `DROP COLUMN`，我们将删除我们刚刚创建的 `email` 列。

```sql
ALTER TABLE users DROP COLUMN email;
```

`DROP COLUMN` 命令将自动删除与我们要删除的列相关的任何约束或索引。因此，如果我们要删除我们的 `email` 列，我们应该会看到该列的唯一性约束也被删除了。我们可以通过再次检查我们的表来验证这一点。

```sql
\d users
```
```
                         Table "public.users"
     Column      |       Type        | Collation | Nullable | Default
-----------------+-------------------+-----------+----------+---------
 name            | character varying |           | not null |
 employee_number | character varying |           | not null |
Indexes:
    "users_employee_number_key" UNIQUE CONSTRAINT, btree (employee_number)
```


## 演示示例

```sql
CREATE TABLE users (
  name varchar not null,
  employee_number varchar not null unique
);

INSERT INTO users VALUES
  ('Sam', 'EM1234'),
  ('Emily', 'EM2345');
```

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第二十八集 [Adding and Dropping Columns](https://www.pgcasts.com/episodes/adding-and-dropping-columns)。
