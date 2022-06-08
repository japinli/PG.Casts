# 修改数据库

大家好，今天我们来看看如何在 PostgreSQL 中修改数据库。

首先，让我们创建一个新数据库。我们将其称为 `example`。

```sql
CREATE DATABASE example;
```

我们可以通过之前介绍的 `list` 元命令来查看数据库列表。

```sql
\l
```
```
                                          List of databases
   Name    | Owner | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider | Access privileges
-----------+-------+----------+---------+---------+------------+-----------------+-------------------
 example   | px    | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 postgres  | px    | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 template0 | px    | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/px            +
           |       |          |         |         |            |                 | px=CTc/px
 template1 | px    | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/px            +
           |       |          |         |         |            |                 | px=CTc/px
(4 rows)
```

想要更改我们的新数据库，我们可以使用 `ALTER DATABASE` 命令。此命令有许多用于更改我们的数据库的选项。你可以通过 `\h ALTER DATABASE` 查看帮助手册。

```sql
\h ALTER DATABASE
```
```
Command:     ALTER DATABASE
Description: change a database
Syntax:
ALTER DATABASE name [ [ WITH ] option [ ... ] ]

where option can be:

    ALLOW_CONNECTIONS allowconn
    CONNECTION LIMIT connlimit
    IS_TEMPLATE istemplate

ALTER DATABASE name RENAME TO new_name

ALTER DATABASE name OWNER TO { new_owner | CURRENT_ROLE | CURRENT_USER | SESSION_USER }

ALTER DATABASE name SET TABLESPACE new_tablespace

ALTER DATABASE name REFRESH COLLATION VERSION

ALTER DATABASE name SET configuration_parameter { TO | = } { value | DEFAULT }
ALTER DATABASE name SET configuration_parameter FROM CURRENT
ALTER DATABASE name RESET configuration_parameter
ALTER DATABASE name RESET ALL

URL: https://www.postgresql.org/docs/devel/sql-alterdatabase.html
```

## RENAME TO

要重命名数据库，我们可以使用 `ALTER DATABASE`，随后跟我们要更改的数据库名称，最后是 `RENAME TO` 选项。

```sql
ALTER DATABASE example RENAME TO test;
```

使用 `list` 元命令，我们可以看到 `example` 数据库现在已重命名为 `test`。
```sql
\l
```
```
                                          List of databases
   Name    | Owner | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider | Access privileges
-----------+-------+----------+---------+---------+------------+-----------------+-------------------
 postgres  | px    | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 template0 | px    | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/px            +
           |       |          |         |         |            |                 | px=CTc/px
 template1 | px    | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/px            +
           |       |          |         |         |            |                 | px=CTc/px
 test      | px    | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
(4 rows)
```

## ALLOW_CONNECTIONS 和 CONNECTION LIMIT

我们可以用 `ALTER DATABASE` 做的另一件事是改变我们的连接设置。首先，让我们尝试删除对我们的 `test` 数据库的连接权限。为此，我们可以使用 `ALTER DATABASE` 和我们的数据库名称，后跟 `ALLOW_CONNECTIONS false`。

```sql
ALTER DATABASE test ALLOW_CONNECTIONS false;
```

要查看我们的数据库是否不再接受连接，我们可以尝试使用 `connect` 元命令（缩写为 `c`）连接它。

```sql
\connect test
```
```
connection to server on socket "/tmp/.s.PGSQL.5432" failed: FATAL:  database "test" is not currently accepting connections
Previous connection kept
```

要允许连接，但对我们的数据库的连接数设置限制，我们再次使用 `ALTER DATABASE`，这次有两个选项：`ALLOW_CONNECTIONS` 和 `CONNECTION LIMIT`。

```sql
ALTER DATABASE test ALLOW_CONNECTIONS true CONNECTION LIMIT 1;
```

我们可以再次使用 `connect` 元命令来查看已为我们的数据库重新启用连接。

```sql
\connect test
```
```
You are now connected to database "test" as user "px".
```

我们还可以通过查看 `pg_database` 表来检查我们的连接限制是否已更新。

```sql
SELECT datname, datconnlimit FROM pg_database;
```
```
  datname  | datconnlimit
-----------+--------------
 postgres  |           -1
 template1 |           -1
 template0 |           -1
 test      |            1
(4 rows)
```

## IS TEMPLATE

和连接选项一样，我们也可以通过 `ALTER DATABASE` 来管理我们的数据库是否是一个模板。这一次，我们使用 `IS_TEMPLATE` 选项。

```sql
ALTER DATABASE test IS_TEMPLATE true;
```

然后我们可以通过查询 `pg_database` 表来检查我们的数据库是否已更新。

```sql
SELECT datname, datistemplate FROM pg_database;
```
```
  datname  | datistemplate
-----------+---------------
 postgres  | f
 template1 | t
 template0 | t
 test      | t
(4 rows)
```

## OWNER TO

最后，让我们使用 `ALTER DATABASE` 命令转移我们数据库的所有权。我们可以通过 `list` 元命令看到我们的 `test` 数据库的当前所有者。

```sql
\l test
```
```
                                       List of databases
 Name | Owner | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider | Access privileges
------+-------+----------+---------+---------+------------+-----------------+-------------------
 test | px    | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
(1 row)
```

要将所有权转移给另一个用户，我们将 `OWNER TO` 选项传递给 `ALTER DATABASE`，然后是我们要转移到的用户的名称。对于这个例子，让我们将 `test` 转移到 `postgres` 用户。

```sql
ALTER DATABASE test OWNER TO postgres;
```

再次通过使用 `list` 元命令，我们可以看到我们的数据库所有者现在已更改。

```sql
\l test
```
```
                                         List of databases
 Name |  Owner   | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider | Access privileges
------+----------+----------+---------+---------+------------+-----------------+-------------------
 test | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
(1 row)
```

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第二十四集 [Altering Databases](https://www.pgcasts.com/episodes/altering-databases)。

[2] 由于本文的输出信息是基于译者的实现环境，因此与原文存在一定差异。
