# Point 数据类型

大家好，今天我们来探讨一下 PostgreSQL 中的 `point` 数据类型。

`Point` 数据类型是 PostgreSQL 几何类型之一，用于表示二维平面上的点。它可以与图形上的 X 和 Y 坐标进行比较。`Point` 是 PostgreSQL 中所有其他几何类型的构建块。

让我们看看 `point` 数据类型是如何工作的。

在此示例中，我们有一个名为 `address` 的表，其中包含一些记录。

```sql
SELECT * FROM addresses;
```
```
 id | street_address |        city        | state |  zip
----+----------------+--------------------+-------+-------
  1 | 320 1st St N   | Jacksonville Beach | FL    | 32250
  2 | 661 W Lake St  | Chicago            | IL    | 60661
(2 rows)
```

如果我们想添加类似地理位置的东西，我们可以使用纬度和经度坐标来表示地址，此时我们可以使用 `point` 数据类型。

我们将首先更改我们的 `addresses` 表并添加地理位置列，将类型设置为 `point`。

```sql
ALTER TABLE addresses ADD COLUMN geolocation point;
```

有了地理位置列之后，我们现在需要添加一些数据。PostgreSQL 中有两种方法可以将数据插入到 `point` 数据类型中。第一种是使用逗号分隔值的字符串。让我们找到 `Jacksonville Breach` 的地址并更新其地理位置。重要的是要注意，当将 `point` 数据类型用于地理定位等数据时，该 `point` 类型希望先接收经度，然后是纬度。

```sql
UPDATE addresses
SET geolocation = '-81.4,30.3'
WHERE city = 'Jacksonville Beach';
```

我们现在可以再次从我们的表中读取，同时可以看到我们的 `Jacksonville Beach` 的地址现在有一个地理位置。

```sql
SELECT * FROM addresses;
```
```
 id | street_address |        city        | state |  zip  | geolocation
----+----------------+--------------------+-------+-------+--------------
  2 | 661 W Lake St  | Chicago            | IL    | 60661 |
  1 | 320 1st St N   | Jacksonville Beach | FL    | 32250 | (-81.4,30.3)
(2 rows)
```
PostgreSQL 中插入 `point` 数据类型的第二中方法是使用 `point` 构造函数。此构造函数接受两个参数，并将它们保存为数据库中 `point` 的值。在这里，我们同样地先提供经度，然后是纬度。

```sql
UPDATE addresses
SET geolocation = point(-87.6, 41.9)
WHERE city = 'Chicago';
```

现在，当我们从数据库中读取数据时，我们可以看到我们的两行都设置了地理位置。

```sql
SELECT * FROM addresses;
```
```
 id | street_address |        city        | state |  zip  | geolocation
----+----------------+--------------------+-------+-------+--------------
  1 | 320 1st St N   | Jacksonville Beach | FL    | 32250 | (-81.4,30.3)
  2 | 661 W Lake St  | Chicago            | IL    | 60661 | (-87.6,41.9)
(2 rows)
```

您还会注意到，我们的地理位置并没有以任何方式被截断。`Point` 数据类型在 PostgreSQL 中保存为两个 `float8` 值或两个双精度浮点数，每个占用 `8` 个字节。这些类型支持最多 `15` 位十进制数字的精度。

查询我们的 `point` 列与 PostgreSQL 中的标准查询略有不同。最明显的例子是尝试根据确切的 `point` 值找到我们的地址。

```sql
SELECT * FROM addresses WHERE geolocation = '-87.6,41.9';
```
```
ERROR:  operator does not exist: point = unknown
LINE 1: SELECT * FROM addresses WHERE geolocation = '-87.6,41.9';
                                                  ^
HINT:  No operator matches the given name and argument types. You might need to add explicit type casts.
```

我们可以从输出中看到，PostgreSQL 告诉我们，我们正在做的操作（将 `point` 类型与未知数据类型进行比较）不存在。我们可以尝试将其强制转换为 `point`，但是您会注意到我们仍然收到错误。

```sql
SELECT * FROM addresses WHERE geolocation = point '-87.6,41.9';
```
```
ERROR:  operator does not exist: point = point
LINE 1: SELECT * FROM addresses WHERE geolocation = point '-87.6,41....
                                                  ^
HINT:  No operator matches the given name and argument types. You might need to add explicit type casts.
```

这次错误告诉我们，我们用来比较 `point` 的运算符不存在。

想要在 PostgreSQL 中使用精确的 `point` 值查找行，我们必须使用支持的几何函数。在我们的例子中，我们将使用 PostgreSQL 定义的 `Same as?` 函数，它是一个波浪号后跟一个等号。

```sql
SELECT * FROM addresses WHERE geolocation ~= '-87.6,41.9';
```
```
 id | street_address |  city   | state |  zip  | geolocation
----+----------------+---------+-------+-------+--------------
  2 | 661 W Lake St  | Chicago | IL    | 60661 | (-87.6,41.9)
(1 row)
```

使用 `Same as?` 运算符，我们可以看到我们终于得到了我们查询的匹配项。您可能还注意到了，我们不必为查询使用 `point` 构造函数；PostgreSQL 能够为我们的查询提供类型转换。



## 演示示例

```sql
CREATE TABLE addresses (
  id serial primary key,
  street_address varchar,
  city varchar,
  state varchar,
  zip varchar
);

INSERT INTO addresses (street_address, city, state, zip) VALUES
  ('320 1st St N', 'Jacksonville Beach', 'FL', '32250'),
  ('661 W Lake St', 'Chicago', 'IL', '60661');
```

## 参考

[1] [Geometry Functions in PostgreSQL](https://www.postgresql.org/docs/current/functions-geometry.html)

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第三十九集 [The Point Data Type](https://www.pgcasts.com/episodes/the-point-data-type)。
