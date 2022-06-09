# 创建和删除表

大家好，今天我们将研究如何在 PostgreSQL 数据库中创建和删除表。

## CREATE TABLE

我们可以通过 `CREATE TABLE` 命令在 PostgreSQL 数据库中创建表。`CREATE TABLE` 命令最基本的用法中需要一个表名，后跟一个逗号分隔的属性列列表。列应采用列名后跟列类型的格式。

```sql
CREATE TABLE test (active boolean, title varchar, data text);
```

在表创建过程中，我们还可以设置主键以及我们可能想要的任何约束。我们将使用编辑器元命令来明确我们将在这里做什么。对于我们的示例，让我们创建一个用户表。

首先，我们要设置一个序列类型的 `id` 列。序列类型是 PostgreSQL 中一种方便的简写。它允许我们创建一个自动递增的整数列。在定义了我们的列类型之后，PostgreSQL 允许我们添加列约束。这里我们要告诉 PostgreSQL `id` 列是我们的主键。

在 `id` 列之后，还有一些其它属性列，所有这些列都有非空约束。对于我们的最后一列，让我们创建一个布尔值，默认值为 `true`。

最后，我们将添加一个表约束，以防止用户使用相同的名字、姓氏和用户名的组合。为此，我们首先指定我们正在创建一个约束，并传入我们正在添加的约束的名称。

如果我们选择不为我们的约束提供名称，PostgreSQL 将自动为我们生成一个。然后我们指定我们正在创建的约束的类型，在这种情况下是一个“唯一”约束，然后是约束将检查的一个或多个列。

```sql
CREATE TABLE users (
  id serial primary key,
  first_name varchar not null,
  last_name varchar not null,
  user_name varchar not null,
  active boolean default true,
  constraint unique_name_user_name unique (first_name, last_name, user_name)
);
```

当我定义完了表之后，我们就可以保存并退出编辑器。我们可以使用 `\d` 元命令来验证我们的表是否正确创建。

```sql
\d users
```
```
                                   Table "public.users"
   Column   |       Type        | Collation | Nullable |              Default
------------+-------------------+-----------+----------+-----------------------------------
 id         | integer           |           | not null | nextval('users_id_seq'::regclass)
 first_name | character varying |           | not null |
 last_name  | character varying |           | not null |
 user_name  | character varying |           | not null |
 active     | boolean           |           |          | true
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
    "unique_name_user_name" UNIQUE CONSTRAINT, btree (first_name, last_name, user_name)
```

如果我们尝试在 PostgreSQL 中创建另一个与现有表同名的表，则会引发错误。这个错误可以通过在 `CREATE TABLE` 命令后使用 `IF NOT EXISTS` 来避免。我们可以通过尝试创建另一个用户表来看到这一点。

```sql
CREATE TABLE users (id serial);
```
```
ERROR:  relation "users" already exists
```

PostgreSQL 给我们一个错误，通知我们该表已经存在。现在让我们再次尝试这个命令，并使用 `IF NOT EXISTS` 选项。

```sql
CREATE TABLE IF NOT EXISTS users (id serial);
```
```
NOTICE:  relation "users" already exists, skipping
CREATE TABLE
```

这次我们看到来自 PostgreSQL 的注释，告诉我们该表已经存在，并且正在跳过创建语句。

## DROP TABLE

要删除一个表，我们使用 `DROP TABLE` 命令，后接我们想要删除的表名。

```sql
DROP TABLE test;
```

我们可以再次使用 `\d` 元命令验证它是否已消失。

```sql
\d test
```
```
Did not find any relation named "test".
```

如果我们尝试删除已经从数据库中删除的表，PostgreSQL 将引发错误。为避免此类错误，`DROP TABLE` 命令具有 `IF EXISTS` 选项。

```sql
DROP TABLE test;
```
```
ERROR:  table "test" does not exist
```
```sql
DROP TABLE IF EXISTS test;
```
```
NOTICE:  table "test" does not exist, skipping
DROP TABLE
```

我们可以从此处的输出中看到，`IF EXISTS` 选项的行为类似于 `CREATE TABLE` 命令中的 `IF NOT EXISTS` 选项。它给了我们一个注释，告诉我们该表不存在，并且跳过当前的 `DROP` 语句。

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第二十七集 [Creating and Dropping Tables](https://www.pgcasts.com/episodes/creating-and-dropping-tables)。
