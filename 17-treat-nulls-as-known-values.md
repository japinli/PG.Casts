# 将 NULL 视为已知值

在这一集中，我们将看看如何使用 `IS DISTINCT FROM` 和它对应的 `IS NOT DISTINCT FROM` 在我们的查询中将空值（`NULL`）视为已知值。

考虑以下场景 - 您有一个 `users` 表，其中包含一个可以为空的字段，用于跟踪用户是否注册了邮件列表<sup>[1]</sup>。

```sql
TABLE users;
```
```
 id | first_name |  last_name  |        email         | subscribed
----+------------+-------------+----------------------+------------
  1 | Jack       | Christensen | jack_c@example.com   | f
  2 | Brian      | Dunn        | brian_d@example.com  | <null>
  3 | Chris      | Erin        | chris_e@example.com  | f
  4 | Dorian     | Karter      | dorian_k@example.com | t
  5 | Joe        | Hashrocket  | joe_h@example.com    | f
  6 | Jane       | Hashrocket  | jane_h@example.com   | <null>
(6 rows)
```

在我们的示例表中，有一些记录的 `subscribed` 为 `true`，这表示他们已经订阅了我们的邮件列表，而 `subscribed = false` 的则表示这些用户还没有订阅。

如果用户还未填写过询问他们是否要订阅的表单，那么 `subscribed` 被设置为空值 `NULL`。在这种情况下，空值对我们有特殊意义，我们希望将其视为已知值。

如果我们想获取所有还没有订阅我们邮件列表的用户，以便我们可以在他们下次登录时向他们显示提示，我们可能会编写如下查询：

```sql
SELECT * FROM users WHERE subscribed <> true;
```

您可能会惊讶于上面的查询结果只会产生以下记录：

```
 id | first_name |  last_name  |        email        | subscribed
----+------------+-------------+---------------------+------------
  1 | Jack       | Christensen | jack_c@example.com  | f
  3 | Chris      | Erin        | chris_e@example.com | f
  5 | Joe        | Hashrocket  | joe_h@example.com   | f
(3 rows)
```

这里发生了什么？默认情况下，相等运算符并不将空值视为已知值，因此假定它不能直接与实际值进行比较。如果要将空值视为已知值，可以像这样显式查询它们：

```sql
SELECT * FROM users WHERE subscribed is null OR subscribed <> true;
```

结果如下所示：

```
 id | first_name |  last_name  |        email        | subscribed
----+------------+-------------+---------------------+------------
  1 | Jack       | Christensen | jack_c@example.com  | f
  2 | Brian      | Dunn        | brian_d@example.com | <null>
  3 | Chris      | Erin        | chris_e@example.com | f
  5 | Joe        | Hashrocket  | joe_h@example.com   | f
  6 | Jane       | Hashrocket  | jane_h@example.com  | <null>
(5 rows)
```

这将起作用，但它不可扩展。当您在一个小查询中执行一次时，它可能没问题，但随着查询变得更大，这会使查询变得混乱且难以阅读。

幸运的是，PostgreSQL 提供了一种更常用的方法来检查一个值是否不等于某个值，包括空值：`is distinct from` 和 `is not distinct from` 查询部分。

我们将示例转换为使用以下语法：

```sql
SELECT * FROM users WHERE subscribed is distinct from true;
```

果然我们得到了正确的结果：

```
 id | first_name |  last_name  |        email        | subscribed
----+------------+-------------+---------------------+------------
  1 | Jack       | Christensen | jack_c@example.com  | f
  2 | Brian      | Dunn        | brian_d@example.com | <null>
  3 | Chris      | Erin        | chris_e@example.com | f
  5 | Joe        | Hashrocket  | joe_h@example.com   | f
  6 | Jane       | Hashrocket  | jane_h@example.com  | <null>
(5 rows)
```

相反，我们可以使用 `is not distinct from` 语法来获取结果等于传递值的记录：

```sql
SELECT * FROM users WHERE subscribed is not distinct from true;
```

此查询将返回所有 `subscribed = true` 的行：

```
 id | first_name | last_name |        email         | subscribed
----+------------+-----------+----------------------+------------
  4 | Dorian     | Karter    | dorian_k@example.com | t
(1 row)
```

同样的语法也适用于查询空值：

```sql
SELECT * FROM users WHERE subscribed is not distinct from null;
```
```
 id | first_name | last_name  |        email        | subscribed
----+------------+------------+---------------------+------------
  2 | Brian      | Dunn       | brian_d@example.com | <null>
  6 | Jane       | Hashrocket | jane_h@example.com  | <null>
(2 rows)
```

当您从您选择的编程语言中向您的SQL查询传递参数时，这种语法就变得特别有用。为了举例说明，我们将使用准备好的语句，这是一种在 PostgreSQL 中重复使用不同参数的 SQL 语句的方法，类似于您的编程语言在幕后所做的事情。

我们将首先测试使用普通相等运算符时会发生什么：

```sql
PREPARE users_with_subscription_status(boolean) AS
  SELECT * FROM users WHERE subscribed = $1;
```

如果我们尝试将空值传递给它，我们刚刚准备的这个查询将不起作用，它也不会将空值视为已知值。

```sql
EXECUTE users_with_subscription_status(null);
```

结果如下所示：

```
 id | first_name | last_name | email | subscribed
----+------------+-----------+-------+------------
(0 rows)
```

现在让我们使用不等号运算符测试相同的查询：

```sql
PREPARE users_without_subscription_status(boolean) AS
  SELECT * FROM users WHERE subscribed <> $1;
```
```
 id | first_name | last_name | email | subscribed
----+------------+-----------+-------+------------
(0 rows)
```

用显式测试空值来重写这些类型的可重用查询真的很困难。相反，如果我们用 `is distinct from` 和 `is not distinct from` 重写它们，它可以完美地工作，首先让我们丢弃之前所有准备好的语句：

```sql
DISCARD ALL;
```

随后使用 `is distinct from` 和 `is not distinct from` 再次创建 `PREPARE`：

```sql
PREPARE users_with_subscription_status(boolean) AS
  SELECT * FROM users WHERE subscribed is not distinct from $1;

PREPARE users_without_subscription_status(boolean) AS
  SELECT * FROM users WHERE subscribed is distinct from $1;
```

这些更新的预处理语句现在支持空值和实际布尔值。让我们测试一下：

```sql
EXECUTE users_without_subscription_status(null);
```
```
 id | first_name |  last_name  |        email         | subscribed
----+------------+-------------+----------------------+------------
  1 | Jack       | Christensen | jack_c@example.com   | f
  3 | Chris      | Erin        | chris_e@example.com  | f
  4 | Dorian     | Karter      | dorian_k@example.com | t
  5 | Joe        | Hashrocket  | joe_h@example.com    | f
(4 rows)
```

```sql
EXECUTE users_without_subscription_status(true);
```
```
 id | first_name |  last_name  |        email        | subscribed
----+------------+-------------+---------------------+------------
  1 | Jack       | Christensen | jack_c@example.com  | f
  2 | Brian      | Dunn        | brian_d@example.com | <null>
  3 | Chris      | Erin        | chris_e@example.com | f
  5 | Joe        | Hashrocket  | joe_h@example.com   | f
  6 | Jane       | Hashrocket  | jane_h@example.com  | <null>
(5 rows)
```

```sql
EXECUTE users_with_subscription_status(true);
```
```
 id | first_name | last_name |        email         | subscribed
----+------------+-----------+----------------------+------------
  4 | Dorian     | Karter    | dorian_k@example.com | t
(1 row)
```

```sql
EXECUTE users_with_subscription_status(null);
```
```
 id | first_name | last_name  |        email        | subscribed
----+------------+------------+---------------------+------------
  2 | Brian      | Dunn       | brian_d@example.com | <null>
  6 | Jane       | Hashrocket | jane_h@example.com  | <null>
(2 rows)
```

## 演示示例

```sql
CREATE TABLE users (
  id serial primary key,
  first_name varchar,
  last_name varchar,
  email varchar not null,
  subscribed boolean
);

INSERT INTO users (first_name, last_name, subscribed, email) VALUES
  ('Jack', 'Christensen', false, 'jack_c@example.com'),
  ('Brian', 'Dunn', null, 'brian_d@example.com'),
  ('Chris', 'Erin', false, 'chris_e@example.com'),
  ('Dorian', 'Karter', true, 'dorian_k@example.com'),
  ('Joe', 'Hashrocket', false, 'joe_h@example.com'),
  ('Jane', 'Hashrocket', null, 'jane_h@example.com');
```

## 译者著

[1] 根据演示示例给出的表结果，这理的顺序有所不同；此外，我通过 `\pset null <null>` 设置了 `NULL` 值的显示。

[2] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第十七集 [Treat Nulls as Known Values](https://www.pgcasts.com/episodes/treat-nulls-as-known-values)。
