# 删除数据库

大家好，今天我们来看看如何在 PostgreSQL 中删除数据库。

首先，让我们创建一个新数据库。我们将其称为 `example`。

```sql
CREATE DATABASE example;
```

我们可以使用 `DROP DATABASE` 命令在 PostgreSQL 中删除数据库。此命令需要我们希望删除的数据库的名称。

```sql
DROP DATABASE example;
```

`DROP DATABASE` 命令有一个 `IF EXISTS` 标志，以防止在我们尝试删除不存在的数据库时可能引发的任何错误。如果我们再次尝试删除 `example` 数据库，我们会看到这样的错误。

```sql
DROP DATABASE example;
```
```
ERROR:  database "example" does not exist
```

但是，如果我们添加 `IF EXISTS` 标志，我们将不再收到错误。

```sql
DROP DATABASE IF EXISTS example;
```
```
NOTICE:  database "example" does not exist, skipping
DROP DATABASE
```

关于删除数据库的最后一个注意事项是不能删除模板数据库。为了探索这种行为，我们首先创建一个模板数据库。

```sql
CREATE DATABASE example IS_TEMPLATE true;
```

当我们尝试删除模版数据库时，我们可以看到 PostgreSQL 抛出了一个错误。

```sql
DROP DATABASE example;
```
```
ERROR:  cannot drop a template database
```

要删除模板数据库，我们必须首先将其更新为非模板数据库。我们可以使用 `ALTER DATABASE` 命令执行此操作，将 `IS_TEMPLATE` 设置为 `false`。

```sql
ALTER DATABASE example IS_TEMPLATE false;
```

我们现在可以再次运行我们的 `DROP DATABASE` 命令以查看它是否成功运行。

```sql
DROP DATABASE example;
```

最后，您同样可以使用 `\h DROP DATABASE` 来查看帮助信息。

```sql
\h DROP DATABASE
```
```
Command:     DROP DATABASE
Description: remove a database
Syntax:
DROP DATABASE [ IF EXISTS ] name [ [ WITH ] ( option [, ...] ) ]

where option can be:

    FORCE

URL: https://www.postgresql.org/docs/devel/sql-dropdatabase.html
```

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第二十五集 [Dropping Databases](https://www.pgcasts.com/episodes/dropping-databases)。

[2] `DROP DATABASE` 命令的帮助信息是基于 PG 15devel 的。
