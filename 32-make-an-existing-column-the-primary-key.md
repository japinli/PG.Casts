# 为列添加主键约束

大家好，今天我们来看看如何为 PostgreSQL 中已存在的列添加主键约束。

在此示例中，我们将处理一个虚拟数据库，该数据库有一个名为 `users` 的表，其中包含 `name` 和 `employee_number` 两列。我们可以通过 `SELECT` 查看该表中已有的数据：

```sql
SELECT * FROM users;
```
```
 name  | employee_number
-------+-----------------
 Sam   | EM1234
 Emily | EM2345
(2 rows)
```

我们还可以通过 `\d` 来检查表当前的属性列以及约束。

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

如您所见，我们目前有一个约束条件，即 `employee_number` 是唯一的。要使现有列成为主键，我们可以使用 `ALTER TABLE` 命令，后接我们需要处理的表，对我们来说是 `users` 表。然后我们指定要进行的操作，即 `ADD PRIMARY KEY`（添加主键），并传递我们需要创建主键的属性列名称。

```sql
ALTER TABLE users ADD PRIMARY KEY (employee_number);
```

我们可以再次查看我们的表结构，发现除了我们已经拥有的唯一性约束之外，`employee_number` 现在还有一个非空约束和一个索引。

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
    "users_pkey" PRIMARY KEY, btree (employee_number)
    "users_employee_number_key" UNIQUE CONSTRAINT, btree (employee_number)
```

重要的是要注意，我们仅能在该列没有空值且没有重复项时，才能将现有的列设为主键。如果列未通过其中任何一项检查，则 `ALTER TABLE` 命令将失败。

## 演示示例

```sql
CREATE TABLE users (
  name varchar not null,
  employee_number varchar not null unique
);

INSERT INTO users values
  ('Sam', 'EM1234'),
  ('Emily', 'EM2345');
```

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第三十二集 [Make an Existing Column the Primary Key](https://www.pgcasts.com/episodes/make-an-existing-column-the-primary-key)。
