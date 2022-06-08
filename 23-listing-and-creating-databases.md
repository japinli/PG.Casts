# 查看和创建数据库

大家好，今天我们来看看如何在 PostgreSQL 中查看和创建数据库。要查看我们当前拥有的所有数据库的列表，我们可以使用 `\list` 元命令，此命令也可以缩写为 `\l`。要创建一个新数据库，我们需要使用 `CREATE DATABASE` 语句后跟我们所需的数据库名称。

```sql
CREATE DATABASE example;
```

在创建新数据库时，我们还可以设置所需的编码、所有者、排序规则和其他一些选项。这些设置选项紧跟在数据库名称之后传递给 `CREATE DATABASE` 命令。尝试在新数据库上使用任何非默认编码设置时需要注意的重要一点是，我们总是必须传递 `template` 选项，并将其设置为 `template0`。这是为了防止从默认的 `template1` 数据库克隆任何潜在的数据损坏。

您可以看到，如果我们不指定模板，PostgreSQL 在尝试创建我们的新数据库时会抛出错误。

```sql
CREATE DATABASE example ENCODING 'sql_ascii' LC_COLLATE 'C';
```
```
ERROR:  new encoding (SQL_ASCII) is incompatible with the encoding of the template database (UTF8)
HINT:  Use the same encoding as in the template database, or use template0 as template.
```

但是，一旦我们传入了 `template`，我们就可以开始了。

```sql
CREATE DATABASE example ENCODING 'sql_ascii' LC_COLLATE 'C' TEMPLATE template0;
```

如果我们再次使用 `\list` 元命令，我们可以看到我们的新数据库现在包含在列表中。

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第二十三集 [Listing and Creating Databases](https://www.pgcasts.com/episodes/listing-and-creating-databases)。
