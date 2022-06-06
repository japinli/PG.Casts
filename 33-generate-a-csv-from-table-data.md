# 从表中导出 CSV

大家好，今天我们来看看如何从 PostgreSQL 的表中导出 CSV 格式的数据。

我们可以使用 `COPY` 命令从 PostgreSQL 中的表导出 CSV 格式的数据。`COPY` 命令需要一个表作为参数，后跟输出的 csv 文件的绝对路径。

在此示例中，我们使用一个带有名为 `users` 的表的虚拟数据库。我们可以通过查看 `users` 表的结构来看到我们的 `users` 有几列。

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

如果我们查询 `users` 表，我们可以看到我们有几行数据。

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

让我们使用 `COPY` 命令将此数据转储到 CSV 中。

```
COPY users
  TO '/Users/marylee/Documents/users.csv';
```

`COPY` 命令运行完成之后，我们可以 `cat /Users/marylee/Documents/users.csv` 来查看我们导出的 CSV 文件。

```bash
cat /Users/marylee/Documents/users.csv
```
```
Sam     sam@example.com 35
Sue     sue@example.com 65
Carl    carl@example.com        45
```

您可能已经注意到我们的 CSV 文件没有标题。要在生成的 CSV 文件中包含标题，您可以在指定输出文件路径后使用 `WITH CSV HEADER` 选项。让我们试试这个。

```sql
COPY USERS
  TO '/Users/marylee/Documents/users.csv'
  WITH CSV HEADER;
```

这次当我们使用 `cat` 查看输出文件时，我们可以看到标题已经添加到顶部了。

```bash
cat /Users/marylee/Documents/users.csv
```
```
name,email,age
Sam,sam@example.com,35
Sue,sue@example.com,65
Carl,carl@example.com,45
```

如果我们只想要表中的特定列，我们可以将它们作为逗号分隔的列表传递到表名之后。在这种情况下，我们只需选择用户的姓名和年龄。

```sql
COPY users (name, age)
  TO '/Users/marylee/Documents/users.csv'
  WITH CSV HEADER;
```

现在，如果我们再次查看 CSV 文件，我们应该会看到我们只有 `users` 表中的姓名和年龄。

```bash
cat /Users/marylee/Documents/users.csv
```
```
name,age
Sam,35
Sue,65
Carl,45
```

`COPY` 命令还支持对正在转储的表的查询。为此，我们将查询包裹在括号中，并将其作为第一个参数传递给复制命令。在这个例子中，我们只转储 40 岁以上的用户。

```sql
COPY (SELECT * FROM users WHERE age > 40)
  TO '/Users/marylee/Documents/users.csv'
  WITH CSV HEADER;
```

```bash
cat /Users/marylee/Documents/users.csv
```
```
name,email,age
Sue,sue@example.com,65
Carl,carl@example.com,45
```

当我们使用查询从表中获取数据时需要注意的一件事

让我们以最后一个例子为例。为了只转储 40 岁以上用户的姓名和电子邮件，我们将 `SELECT` 语句更改为只选择我们想要的列。

```sql
COPY (SELECT name, email FROM users WHERE age > 40)
  TO '/Users/marylee/Documents/users.csv'
  WITH CSV HEADER;
```

如果我们再次查看 CSV 文件，我们可以看到仅包含 40 岁以上用户的姓名和电子邮件。

```bash
\! cat /Users/marylee/Documents/users.csv
```
```
name,email
Sue,sue@example.com
Carl,carl@example.com
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

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第三十三集 [Generate a CSV from Table Data](https://www.pgcasts.com/episodes/generate-a-csv-from-table-data)。
