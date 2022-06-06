# 使用 PostGIS 的地理定位

大家好，今天我们来看看如何使用 PostGIS 扩展来处理 PostgreSQL 中的地理位置。

PostGIS 是 PostgreSQL 的第三方扩展，可以使用 PostgreSQL 中的 `CREATE EXTENSION` 命令添加。

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

在使用地理定位时，您通常需要处理距离测量，PostGIS 建议使用 `geography` 数据类型。这种类型比 `geometry` 类型慢，可用的功能更少，但它不需要我们了解投影和平面坐标系。让我们在 `addresses` 表中添加 `geography` 列，然后看看我们如何处理数据。

在此示例中，我们有一个名为 `addresses` 的表，其中包含一些记录。

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

## 创建列

当我们增加了新的 `geography` 列，我们现在需要添加数据。我们有几种方法可以做到这一点。

第一种方式是使用字符串，因此我们将更新地址并将地理位置设置为特定格式的字符串，PostGIS 将其称为众所周知的文本表示或 WKT。在这种情况下，我们的 WKT 以我们要插入的数据类型开始，即 `point`，然后是括号和两个坐标值。在经纬度坐标方面，PostGIS 的 `point` 遵循与 Postgres 的 `point` 相同的规则；他们都必须先是经度，然后是纬度。

```sql
UPDATE addresses SET geolocation = 'point(30 -81)' WHERE id = 1;
```

现在，我们可以从 `addresses` 表中查找我们刚刚尝试更新的行，并查看其地理位置的值。

```sql
SELECT * FROM addresses WHERE id = 1;
```
```
```

第二种方式是使用 PostGIS 提供的 `ST_MakePoint()` 函数来更新地理位置列。这个函数接受我们的经度和纬度坐标作为参数，同样地是经度优先，并将它们保存为数据库中的一个 `point` 中。

```sql
UPDATE addresses SET geolocation = ST_MakePoint(longitude, latitude);
```

`ST_MakePoint()` 函数需要注意的一点是它不会自动知道我们正在寻找的 SRID。在我们的最后一个示例中，因为我们正在插入数据库并且该列设置了 SRID，所以我们被覆盖了。

但是，如果我们尝试使用 `ST_MakePoint()` 比较或选择任意点值，则 SRID 将是未知的。

```sql
SELECT ST_MakePoint(-80, 30);
```
```
```

在这种情况下，我们需要将 `ST_MakePoint()` 函数包裹在 `ST_SetSRID()` 函数中，并明确告诉 PostGIS 我们打算使用的 SRID。

```sql
SELECT ST_SetSRID(ST_MakePoint(-80, 30), 4326);
```

## 插入数据

## 计算距离

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
## PostGIS

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第四十二集 [Geolocations Using PostGIS](https://www.pgcasts.com/episodes/geolocations-using-postgis)。
