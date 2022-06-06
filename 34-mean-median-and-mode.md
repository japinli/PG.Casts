# Mean、Median 和 Mode

大家好，今天我们来看看如何从 PostgreSQL 的列数据中求均值（`mean`）、中位数（`median`）和众数（`mode`）。

## 均值

要获取列的平均值，我们可以使用 `avg` 聚合函数。此函数需要我们尝试获取平均值的列的名称。

```sql
SELECT avg(age) FROM people;
```
```
         avg
---------------------
 35.5000000000000000
(1 row)
```

## 众数

要获取列的众数，即出现次数最多的数，我们可以使用有序集合聚合函数 `mode()`。此聚合函数需要指定一个有序组才能正常工作。在我们的例子中，我们正在寻找我们数据组中最常见的年龄，所以我们将传递 `within group`，然后是 `order by age`。

```sql
SELECT mode() within group (order by age) FROM people;
```
```
 mode
------
   34
(1 row)
```

`mode()` 聚合函数需要注意的一点是，如果有多个相同的结果出现，则只返回第一个。

## 中位数

为了从我们的数据集中获取中位数，我们不会使用聚合函数。相反，我们将使用 `OFFSET` 子句。由于我们将使用 `OFFSET` 子句，因此按年龄对数据进行排序很重要，这样我们才能获得真正的中位数。

我们能使用 `OFFSET` 子句找到中位数，是因为它可以接受子查询作为其逻辑的一部分。这意味着我们可以使用带有子查询的偏移量来查找表中条目的总数，然后将其分成两半以获取数据集的中位数。

```sql
SELECT age
FROM people
ORDER BY age
OFFSET (select count(*) from people) / 2
LIMIT 1;
```
```
 age
-----
  34
(1 row)
```

除了上述方法之外，我们还可以使用 `percentile_disc()` 有序集聚合函数来计算中位数，如下所示：

```sql
SELECT percentile_disc(0.5) within group (order by age) FROM people;
```
```
 percentile_disc
-----------------
              34
(1 row)
```

## 演示示例

```sql
CREATE TABLE people (id serial primary key, name varchar, age integer);

INSERT INTO people (name, age) VALUES
  ('Sam', 23), ('Sue', 34), ('Fred', 45), ('Emily', 56), ('Sally', 21), ('John', 34);
```

## 参考

[1] https://www.postgresql.org/docs/current/functions-aggregate.html

[2] https://www.educba.com/postgresql-median/

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第三十四集 [Mean, Median, and Mode](https://www.pgcasts.com/episodes/mean-median-and-mode)。

[2] 本文中的 `percentile_disc()` 方式求解中位数原文中并未提及，该方法来源于参考中的文章。
