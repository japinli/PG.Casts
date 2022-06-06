# 使用 earthdistance 和 point 的地理定位

大家好，今天我们来看看如何使用 PostgreSQL 中的 `earthdistance` 模块和 `point` 数据类型来处理地理位置。

在使用纬度和经度地理位置的数据库中的一个常见的需求是查询在某个半径范围内的位置或与另一个位置的距离。有多种方法可以轻松管理此类查询。您可以将 PostgreSQL 中的 `earthdistance` 模块与 `point` 或 `cube` 一起使用，也可以使用第三方 PostGIS 扩展。在这一集中，我们将重点关注将 `earthdistance` 模块与 `point` 数据类型一起使用。

`Earthdistance` 模块是官方支持的 PostgreSQL 扩展，必须手动启用。这里最为重要的时，当我们将其与 `point` 数据类型一起使用时，它还依赖于 `cube` 扩展。我将快速安装两者。

```sql
CREATE EXTENSION IF NOT EXISTS cube;
CREATE EXTENSION IF NOT EXISTS earthdistance;
```

对于这个例子，我们将使用 `addresses` 表，它有一个类型为 `point` 的地理位置列和一些记录。

```sql
SELECT * FROM addresses;
```
```
 id |        name        |  street_address   |        city        | state |  zip  |       geolocation
----+--------------------+-------------------+--------------------+-------+-------+--------------------------
  1 | Hashrocket JAX     | 320 1st St N      | Jacksonville Beach | FL    | 32250 | (-81.3927381,30.2918842)
  2 | Hashrocket Chicago | 661 W Lake St     | Chicago            | IL    | 60661 | (-87.6473133,41.8853881)
  3 | Satchel's Pizza    | 1800 NE 23rd Ave  | Gainesville        | FL    | 32609 | (-82.3018702,29.6739466)
  4 | V Pizza            | 528 1st St N      | Jacksonville Beach | FL    | 32250 | (-81.3905175,30.2938423)
  5 | Artichoke Pizza    | 321 E 14th St     | New York           | NY    | 10003 | (-73.9860525,40.7321652)
  6 | Giordano's         | 130 E Randolph St | Chicago            | IL    | 60601 | (-87.6252984,41.8850284)
(6 rows)
```

我们将通过两个示例来了解如何使用 `earthdistance` 模块。首先是确定位置之间的距离。第二个是在另一个位置的某个半径范围内寻找位置。

让我们从计算两个位置之间的距离开始。我将使用 `\e` 元命令来使用 vim 作为我的查询编辑器。

要计算 `Hashrocket Jacksonville` 办公室与我们表中所有其他地址之间的距离，我们将从 `addresses` 中进行选择并进行横向连接以获取 `Hashrocket Jacksonville` 地址与其它位置进行比较。通过该操作，我们可以使用 `earthdistance` 模块提供给我们的 `<@>` 运算符来完成我们的查询语句以确定 `Hashrocket Jacksonville` 地理位置和我们其它地理位置之间的距离。该运算符期望得到两个点进行比较，并返回两点之间的距离（以英里为单位）。

```sql
SELECT
  name,
  (a.geolocation<@>hr_jax.geolocation) as distance
FROM
  addresses a,
  lateral (select id, geolocation from addresses where name = 'Hashrocket JAX') as hr_jax
WHERE
  a.id <> hr_jax.id
ORDER BY
  distance;
```
```
        name        |      distance
--------------------+--------------------
 V Pizza            | 0.1893526586258982
 Satchel's Pizza    |  69.16002814082835
 Artichoke Pizza    |  832.2112578664457
 Giordano's         |  872.6949011564224
 Hashrocket Chicago |   873.203527399339
(5 rows)
```

我们可以从我们的输出中看到，我们现在有了每个地址与 `Hashrocket Jacksonville` 办公室的距离。

如果仅查找距 `Jacksonville` 办公室一定距离内的位置，我们可以快速地对现有查询进行修改。由于我们已经知道如何获得两点之间的距离，我们可以重用相同的逻辑，现在将其作为 `where` 子句的一部分，并在不等式中使用它来检查位置之间的距离是否小于特定值，例如 100 英里。

```sql
SELECT
  name,
  (a.geolocation<@>hr_jax.geolocation) as distance
FROM
  addresses a,
  lateral (select geolocation, id from addresses where name = 'Hashrocket JAX') as hr_jax
WHERE
  a.id <> hr_jax.id
  AND (a.geolocation<@>hr_jax.geolocation) < 100
ORDER BY
  distance;
```
```
      name       |      distance
-----------------+--------------------
 V Pizza         | 0.1893526586258982
 Satchel's Pizza |  69.16002814082835
(2 rows)
```

通过这种更改，我们可以从输出中看到，现在，我们只包括距离 `Hashrocket Jacksonville` 不到 100 英里的地址。

`Earthdistance` 模块和带有 `point` 的模块的使用还有有一些注意事项。

首先，`point` 的距离计算在两极和 180 度经线上将失效。这可能是一个问题，具体取决于您的用例。如果您试图为用户寻找附近的披萨店，那么您很可能不是在处理北极和南极或俄罗斯的最东北端。但是，如果您不想让自己面对潜在的问题，那么建议您使用基于 `cube` 的地球距离计算。

其次，重要的是要记住，地理位置的 `point` 的值需要保存为“经度、纬度”。

最后，PostgreSQL 的 `earthdistance` 模块假设地球是一个完美的球体，这并不完全准确。如果您在处理地理位置时需要极高的准确性，PostgreSQL 建议您考虑使用 PostGIS 扩展。

## 演示示例

```sql
CREATE TABLE addresses (
  id serial primary key,
  name varchar,
  street_address varchar,
  city varchar,
  state varchar,
  zip varchar,
  geolocation point
);

INSERT INTO addresses (name, street_address, city, state, zip, geolocation) VALUES
  ('Hashrocket JAX', '320 1st St N', 'Jacksonville Beach', 'FL', '32250', '-81.3927381,30.2918842'),
  ('Hashrocket Chicago', '661 W Lake St', 'Chicago', 'IL', '60661', '-87.6473133,41.8853881'),
  ('Satchel''s Pizza', '1800 NE 23rd Ave', 'Gainesville', 'FL', '32609', '-82.3018702,29.6739466'),
  ('V Pizza', '528 1st St N', 'Jacksonville Beach', 'FL', '32250', '-81.3905175,30.2938423'),
  ('Artichoke Pizza', '321 E 14th St', 'New York', 'NY', '10003', '-73.9860525,40.7321652'),
  ('Giordano''s', '130 E Randolph St', 'Chicago', 'IL', '60601', '-87.6252984,41.8850284');
```

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第四十集 [Geolocations Using Earthdistance and Points](https://www.pgcasts.com/episodes/geolocations-using-earthdistance-and-points)。
