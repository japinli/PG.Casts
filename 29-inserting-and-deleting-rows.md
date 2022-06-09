#  插入和删除行

大家好，今天我们来看看如何在 PostgreSQL 中插入和删除行。

对于这个示例，我已经创建了一个带有名为 `users` 表的虚拟数据库。我们可以使用 `\d` 元命令来查看表结构。

```sql
\d users
```
```
                     Table "public.users"
  Column  |       Type        | Collation | Nullable | Default
----------+-------------------+-----------+----------+---------
 name     | character varying |           |          |
 age      | integer           |           |          |
 hometown | character varying |           |          |
```

## INSERT

要向我们的 `users` 表添加新行，我们可以使用 `INSERT` 命令。`INSERT` 命令需要一个表名和一个逗号分隔的数据值列表。

```sql
INSERT INTO users VALUES ('Bob', 35, 'New York');
```

默认情况下，如果没有为插入命令提供列名列表，则提供的值会按照声明的顺序插入列中。如果我们想改变顺序或省略列，我们需要在 `INSERT` 命令中指定我们想要插入的列名称。

```sql
INSERT INTO users (age, hometown) VALUES (35, 'Houston');
```

我们还可以通过提供逗号分隔的列值列表来一次插入多行。

```sql
INSERT INTO users VALUES ('Joe', 42, 'Tampa'), ('Amy', 24, 'Boston');
```

在插入时，PostgreSQL 将尝试对所有值与预期类型不匹配的列自动进行类型转换。例如，让我们将一个字符串传递给我们的整数列。

```sql
INSERT INTO users VALUES ('Sam', '33', 'Detroit');
```

通过查询我们的新用户，我们可以看到年龄已正确转换为整数。

```sql
SELECT * FROM users WHERE name = 'Sam';
```
```
 name | age | hometown
------+-----+----------
 Sam  |  33 | Detroit
(1 row)
```

如果 POstgreSQL 不能自动转换我们列值，它会抛出一个错误。

```sql
INSERT INTO users VALUES ('Sam', 'Some age', 'Detroit');
```
```
ERROR:  invalid input syntax for type integer: "Some age"
LINE 1: INSERT INTO users VALUES ('Sam', 'Some age', 'Detroit');
                                         ^
```

## RETURNING

PostgreSQL 很容易让我们使用非标准的 `RETURNING` 扩展来查看插入到表中的数据。我们可以通过将 `RETURNING` 和我们想要的列名（或者只是 `*` 用于表示所有列）附加到我们的 `INSERT` 命令后来使用此扩展，如下所示：

```sql
INSERT INTO users VALUES ('Sue', 89, 'Phoenix') RETURNING *;
```
```
 name | age | hometown
------+-----+----------
 Sue  |  89 | Phoenix
(1 row)

INSERT 0 1
```

## DELETE

要从我们的表中删除行，我们可以使用 `DELETE` 命令。`DELETE` 命令允许我们使用 `WHERE` 子句来选择要删除的行。因此，如果我们想删除所有名为 `Sam` 的用户，我们可以这样做：

```sql
DELETE FROM users WHERE name = 'Sam';
```

我们之前使用的 `RETURNING` 扩展也可以用在 `DELETE` 命令上，这样我们就可以看到被删除的行。让我们使用它并尝试删除所有 50 岁以上的用户。

```sql
DELETE FROM users WHERE age > 50 RETURNING *;
```
```
 name | age | hometown
------+-----+----------
 Sue  |  89 | Phoenix
(1 row)

DELETE 1
```

如果我们的目标是从表中完全删除所有用户，我们只需从 `DELETE` 语句中省略 `WHERE` 子句即可<sup>[1]</sup>。

```sql
DELETE FROM users;
```

如果我们现在尝试从表中提取数据，我们可以看到它是空的。

```sql
SELECT * FROM users;
```
```
 name | age | hometown
------+-----+----------
(0 rows)
```

## 演示示例

```sql
CREATE TABLE users (
  name varchar,
  age integer,
  hometown varchar
);
```

## 译者著

[1] 我们也可以通过 `TRUNCATE` 的方式来删除表中的所有记录，`DELETE` 是标记为删除，`TRUNCATE` 则是将整个数据文件清空。

[2] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第二十九集 [Inserting and Deleting Rows](https://www.pgcasts.com/episodes/inserting-and-deleting-rows)。
