# 使用 Earthdistance 和 Cube 的地理定位

大家好，今天我们将看看如何使用 PostgreSQL 中的 `earthdistance` 模块和基于 `cube` 的地球距离来处理地理定位。

在使用纬度和经度地理位置的数据库中的一个常见的需求是查询在某个半径范围内的位置或与另一个位置的距离。有多种方法可以轻松管理此类查询。您可以将 PostgreSQL 中的 `earthdistance` 模块与 `point` 或 `cube` 一起使用，也可以使用第三方 PostGIS 扩展。在这一集中，我们将重点关注将 `earthdistance` 模块与 `cube` 模块一起使用<sup>[1]</sup>。

`Earthdistance` 模块是官方支持的 PostgreSQL 扩展，必须手动启用。这里最为重要的时，当我们将其与 `point` 数据类型一起使用时，它还依赖于 `cube` 扩展。我将快速安装两者。

```sql
CREATE EXTENSION IF NOT EXISTS cube;
CREATE EXTENSION IF NOT EXISTS earthdistance;
```

对于此示例，我们将使用 `addresses` 表，该表具有 `float8` 或双精度类型的纬度和经度列，以及一些记录。

```sql
\d addresses
```
```
                                     Table "public.addresses"
     Column     |       Type        | Collation | Nullable |                Default
----------------+-------------------+-----------+----------+---------------------------------------
 id             | integer           |           | not null | nextval('addresses_id_seq'::regclass)
 name           | character varying |           |          |
 street_address | character varying |           |          |
 city           | character varying |           |          |
 state          | character varying |           |          |
 zip            | character varying |           |          |
 latitude       | double precision  |           |          |
 longitude      | double precision  |           |          |
Indexes:
    "addresses_pkey" PRIMARY KEY, btree (id)
```

```sql
SELECT * FROM addresses;
```
```
 id |        name        |  street_address   |        city        | state |  zip  |  latitude  |  longitude
----+--------------------+-------------------+--------------------+-------+-------+------------+-------------
  1 | Hashrocket JAX     | 320 1st St N      | Jacksonville Beach | FL    | 32250 | 30.2918842 | -81.3927381
  2 | Hashrocket Chicago | 661 W Lake St     | Chicago            | IL    | 60661 | 41.8853881 | -87.6473133
  3 | Satchel's Pizza    | 1800 NE 23rd Ave  | Gainesville        | FL    | 32609 | 29.6739466 | -82.3018702
  4 | V Pizza            | 528 1st St N      | Jacksonville Beach | FL    | 32250 | 30.2938423 | -81.3905175
  5 | Artichoke Pizza    | 321 E 14th St     | New York           | NY    | 10003 | 40.7321652 | -73.9860525
  6 | Giordano's         | 130 E Randolph St | Chicago            | IL    | 60601 | 41.8850284 | -87.6252984
(6 rows)
```

我们将看两个示例，了解如何在 `earthdistance` 模块中使用基于 `cube` 的计算。首先是确定位置之间的距离。第二个是在另一个位置的某个半径范围内寻找位置。

让我们从计算两个位置之间的距离开始。我将使用 `\e` 元命令来使用 vim 作为我的查询编辑器。

要计算 `Hashrocket Jacksonville` 办公室与我们表中所有其他地址之间的距离，我们将从 `addresses` 中进行选择并进行横向连接以获取 `Hashrocket Jacksonville` 地址与其它位置进行比较。通过该操作，我们可以完成我们的查询语句以确定 `Hashrocket Jacksonville` 地理位置与我们其他地理位置之间的距离。

```sql
SELECT
  name,
  earth_distance(
    ll_to_earth(a.latitude, a.longitude),
	ll_to_earth(hr_jax.latitude, hr_jax.longitude)
  ) as distance
FROM
  addresses a,
  lateral (select id, latitude, longitude from addresses where name = 'Hashrocket JAX') as hr_jax
WHERE
  a.id <> hr_jax.id
ORDER BY
  distance;
```
```
        name        |      distance
--------------------+--------------------
 V Pizza            |  305.0770482495325
 Satchel's Pizza    | 111427.73170012432
 Artichoke Pizza    | 1340823.8147407905
 Giordano's         | 1406049.3599585204
 Hashrocket Chicago | 1406868.8371920458
(5 rows)
```

我们可以从我们的输出中看到，我们现在有了每个地址到 `Hashrocket Jacksonville` 办公室的距离（以米为单位）。如果我们想以英里为单位，我们可以对 `earth_distance` 进行快速除法，使用米到英里的转换值，即 `1609.344`。

```sql
SELECT
  name,
  earth_distance(
    ll_to_earth(a.latitude, a.longitude),
	ll_to_earth(hr_jax.latitude, hr_jax.longitude)
  ) / 1609.344 as distance
FROM
  addresses a,
  lateral (select id, latitude, longitude from addresses where name = 'Hashrocket JAX') as hr_jax
WHERE
  a.id <> hr_jax.id
ORDER BY
  distance;
```
```
        name        |      distance
--------------------+---------------------
 V Pizza            | 0.18956608919505866
 Satchel's Pizza    |   69.23798249480802
 Artichoke Pizza    |   833.1492923456952
 Giordano's         |   873.6785671419661
 Hashrocket Chicago |   874.1877666875731
(5 rows)
```

如果仅查找距 `Jacksonville` 办公室一定距离内的位置，我们可以快速地对现有查询进行修改。由于我们已经知道如何获得两个位置之间的地球距离，我们现在可以重用相同的逻辑，将其作为 `where` 子句的一部分，并在不等式中使用它来检查距离是否小于特定值，例如 100 英里，这意味着我们可以从之前的转换中移动小数点。

```sql
SELECT
  name,
  earth_distance(
    ll_to_earth(a.latitude, a.longitude),
	ll_to_earth(hr_jax.latitude, hr_jax.longitude)
  ) / 1609.344 as distance
FROM
  addresses a,
  lateral (select id, latitude, longitude from addresses where name = 'Hashrocket JAX') as hr_jax
WHERE
  a.id <> hr_jax.id
  AND earth_distance(
    ll_to_earth(a.latitude, a.longitude),
	ll_to_earth(hr_jax.latitude, hr_jax.longitude)
  ) < 160934.4
ORDER BY
  distance;
```
```
      name       |      distance
-----------------+---------------------
 V Pizza         | 0.18956608919505866
 Satchel's Pizza |   69.23798249480802
(2 rows)
```

通过这种更改，我们可以从输出中看到，现在，我们只包括距离 `Hashrocket Jacksonville` 不到 100 英里的地址。

重要的是要记住 PostgreSQL 的 `earthdistance` 模块假设地球是一个完美的球体，这并不完全准确。如果您在处理地理位置时需要极高的准确性，PostgreSQL 建议您考虑使用 PostGIS 扩展。

## 演示示例

```sql
CREATE TABLE addresses (
  id serial primary key,
  name varchar,
  street_address varchar,
  city varchar,
  state varchar,
  zip varchar,
  latitude float8,
  longitude float8
);

INSERT INTO addresses (name, street_address, city, state, zip, longitude, latitude) VALUES
  ('Hashrocket JAX', '320 1st St N', 'Jacksonville Beach', 'FL', '32250', '-81.3927381' ,'30.2918842'),
  ('Hashrocket Chicago', '661 W Lake St', 'Chicago', 'IL', '60661', '-87.6473133', '41.8853881'),
  ('Satchel''s Pizza', '1800 NE 23rd Ave', 'Gainesville', 'FL', '32609', '-82.3018702', '29.6739466'),
  ('V Pizza', '528 1st St N', 'Jacksonville Beach', 'FL', '32250', '-81.3905175', '30.2938423'),
  ('Artichoke Pizza', '321 E 14th St', 'New York', 'NY', '10003', '-73.9860525', '40.7321652'),
  ('Giordano''s', '130 E Randolph St', 'Chicago', 'IL', '60601', '-87.6252984', '41.8850284');
```

## 译者著

[1] 原文此处写的是 `points`，应该是笔误。

[2] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第四十一集 [Geolocations Using Earthdistance and Cubes](https://www.pgcasts.com/episodes/geolocations-using-earthdistance-and-cubes)。
