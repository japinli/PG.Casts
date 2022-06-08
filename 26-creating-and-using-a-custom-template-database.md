# 创建和使用自定义模版数据库

大家好，今天我们来看看如何在 PostgreSQL 中创建和使用自定义模板数据库。要创建一个新的模板数据库，我们可以使用 `CREATE DATABASE` 命令，并指定 `IS_TEMPLATE = true` 选项。如果我们想要在新的自定义模板上自定义语言环境或编码，我们应该确保我们是从 PostgreSQL 的 `template0` 模版克隆，而不是默认的 `template1`。

```sql
CREATE DATABASE example IS_TEMPLATE true ENCODING 'SQL_ASCII' TEMPLATE template0;
```

我们可以使用 `list` 命令查看我们的新模板数据库。

```sql
\l
```
```
                                          List of databases
   Name    | Owner | Encoding  | Collate |  Ctype  | ICU Locale | Locale Provider | Access privileges
-----------+-------+-----------+---------+---------+------------+-----------------+-------------------
 example   | px    | SQL_ASCII | C.UTF-8 | C.UTF-8 |            | libc            |
 postgres  | px    | UTF8      | C.UTF-8 | C.UTF-8 |            | libc            |
 template0 | px    | UTF8      | C.UTF-8 | C.UTF-8 |            | libc            | =c/px            +
           |       |           |         |         |            |                 | px=CTc/px
 template1 | px    | UTF8      | C.UTF-8 | C.UTF-8 |            | libc            | =c/px            +
           |       |           |         |         |            |                 | px=CTc/px
(4 rows)
```

我们可以通过检查 `pg_database` 数据来验证它是否是一个模板数据库。

```sql
SELECT datistemplate FROM pg_database WHERE datname = 'example';
```
```
 datistemplate
---------------
 t
(1 row)
```

要创建从我们的自定义模板数据库克隆的新数据库，我们需要使用 `CREATE DATABASE` 命令，并将我们的模板名称传递给 `TEMPALTE` 选项。

```sql
CREATE DATABASE test TEMPLATE example;
```

再次使用 `list` 元命令，我们可以看到我们的新数据库具有自定义模板中的 `SQL_ASCII` 编码。

```sql
\l
```
```
                                          List of databases
   Name    | Owner | Encoding  | Collate |  Ctype  | ICU Locale | Locale Provider | Access privileges
-----------+-------+-----------+---------+---------+------------+-----------------+-------------------
 example   | px    | SQL_ASCII | C.UTF-8 | C.UTF-8 |            | libc            |
 postgres  | px    | UTF8      | C.UTF-8 | C.UTF-8 |            | libc            |
 template0 | px    | UTF8      | C.UTF-8 | C.UTF-8 |            | libc            | =c/px            +
           |       |           |         |         |            |                 | px=CTc/px
 template1 | px    | UTF8      | C.UTF-8 | C.UTF-8 |            | libc            | =c/px            +
           |       |           |         |         |            |                 | px=CTc/px
 test      | px    | SQL_ASCII | C.UTF-8 | C.UTF-8 |            | libc            |
(5 rows)
```

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第二十六集 [Creating and Using a Custom Template Database](https://www.pgcasts.com/episodes/creating-and-using-a-custom-template-database)。

[2] 本文的输出基于 PG 15devel 版本。
