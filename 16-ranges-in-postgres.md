# Postgres 中的 range 类型

在这一集中，我们将研究 PostgreSQL 中的范围（`range`）类型以及它们如何简化查询和提高数据完整性。范围类型表示值的范围，例如 1 到 5 或 1 月 10 日到 1 月 12 日。在没有范围类型的数据库中，有必要将范围表示为两列，例如 `start_date` 和 `end_date`。虽然这可以工作，但也很难使用。 PostgreSQL 范围类型极大地简化了这一点。让我们看看我们通过范围获得的一些特性。

首先，让我们使用范围构造函数来构建范围。

```sql
SELECT int4range(1, 5);
```
```
 int4range
-----------
 [1,5)
(1 row)
```

要查找某个值是否包含在某个范围内，可以使用 `@>` 包含运算符。

```sql
SELECT int4range(1, 5) @> 3;
```
```
 ?column?
----------
 t
(1 row)
```

```sql
SELECT int4range(1, 5) @> 6;
```
```
 ?column?
----------
 f
(1 row)
```

我们还可以检查范围是否重叠。

```sql
SELECT int4range(1, 5) && int4range(3, 7);
```
```
 ?column?
----------
 t
(1 row)
```

```sql
SELECT int4range(1, 5) && int4range(6, 7);
```
```
 ?column?
----------
 f
(1 row)
```

但是范围的界限呢？它们是包含边界还是不包含边界呢？

```sql
SELECT int4range(1, 5) @> 1;
```
```
 ?column?
----------
 t
(1 row)

```

从上面可以看出，下边界是包含的。

```sql
SELECT int4range(1, 5) @> 5;
```
```
 ?column?
----------
 f
(1 row)
```

而上边界是不包含的。这可以通过将第三个参数传递给范围构造函数来更改。第三个参数是一个双字符字符串，第一个字符代表下限类型，第二个字符代表上限类型。方括号（`[` 和 `]`）表示包含，括号（`(` 和 `)`）表示不包含。

```sql
SELECT int4range(1, 5, '[]') @> 5;
```
```
 ?column?
----------
 t
(1 row)
```

默认的包含下边界而不包含上边界通常是您想要的。这在测试重叠时效果很好，因为它让范围只是相接而不重叠。

```sql
SELECT int4range(1, 5) && int4range(5, 7);
```
```
 ?column?
----------
 f
(1 row)
```

范围也可以通过使用 `NULL` 作为边之一来设置为无界。

```sql
SELECT int4range(3, NULL) @> 42;
```
```
 ?column?
----------
 t
(1 row)
```

使用一个范围而不是两个单独的列还可以提供额外的数据完整性，因为它可以防止上下边界颠倒的范围。

```sql
SELECT int4range(5, 3);
```
```
ERROR:  range lower bound must be less than or equal to range upper bound
```

此外，PostgreSQL 支持可以防止存储重叠范围的约束。假设我们正在构建一个数据库来处理酒店预订。一个简单的预订表将包含房间和日期范围。

```sql
CREATE TABLE reservations(
  id serial primary key,
  room varchar not null,
  dates daterange not null,
  exclude using gist (room with =, dates with &&)
);
```

下面将展示如何解释这个约束。第一个词 `exclude` 表示排除任何行与任何其他行匹配。`using gist` 意味着底层索引类型是 `GiST`，它代表广义搜索树。括号中的表达式是匹配条件。首先使用等号运算符（`=`）检查房间列。然后使用重叠运算符（`&&`）检查日期。

哎呀。 PostgreSQL 给我们一个错误。

```
ERROR:  data type character varying has no default operator class for access method "gist"
HINT:  You must specify an operator class for the index or define a default operator class for the data type.
```

问题是 `GiST` 索引没有为 `varchar` 定义 `=` 运算符。我们需要安装 btree_gist 扩展。

```sql
CREATE EXTENSION btree_gist;
CREATE TABLE reservations(
  id serial primary key,
  room varchar not null,
  dates daterange not null,
  exclude using gist (room with =, dates with &&)
);
```

`btree_gist` 这个扩展为 `GiST` 索引定义了许多的操作符。现在让我们试试这个。首先，我们将为房间 101 插入一行。

```sql
INSERT INTO reservations(room, dates) VALUES  ('101', daterange('2016-11-01', '2016-11-10'));
```

我们可以为不同的房间插入具有重叠日期的行<sup>[1]</sup>。

```sql
INSERT INTO reservations(room, dates) VALUES  ('201', daterange('2016-11-05', '2016-11-15'));
```

但是我们不能为同一个房间插入具有重叠日期的行。

```sql
INSERT INTO reservations(room, dates) VALUES  ('101', daterange('2016-11-07', '2016-11-15'));
```
```
ERROR:  conflicting key value violates exclusion constraint "reservations_room_dates_excl"
DETAIL:  Key (room, dates)=(101, [2016-11-07,2016-11-15)) conflicts with existing key (room, dates)=(101, [2016-11-01,2016-11-10)).
```

排除约束不仅可以提高数据完整性，而且在使用重叠运算符进行搜索时，底层索引也可以显着提高性能。

## 译者著

[1] 原文这里的房间号是 `101`，应属于笔误，视频里面使用的是 `201`。

[2] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第十六集 [Ranges in Postgres](https://www.pgcasts.com/episodes/ranges-in-postgres)。
