# Postgres 9.6 新特性之显示和编辑视图

在这一集中，我们将研究 PostgreSQL 9.6 中用于显示和编辑视图的两个 psql 新命令。如果您还没有看过我们对数据库视图的介绍，请查看[此处](12-intro-to-views.md)。回顾一下，视图是一种抽象，它将复杂查询的逻辑封装在一个简单的接口后面。今天我们将研究 PostgreSQL 9.6 中的两个新命令，它们可以帮助我们更好地理解和改变我们的视图。

为了做好准备，我已经建立了一个示例数据库，其中包含两个表，`employees` 和 `hometowns`，以及一个连接它们的名为 `employee_hometowns` 的视图。该脚本包含在下面。

我们可以想视同其他表一样通过 `SELECT` 查看视图中的记录。

```sql
SELECT * FROM employee_hometowns;
```
```
    full_name    |                                 title                                  |    hometown    | state
-----------------+------------------------------------------------------------------------+----------------+-------
 Liz Lemon       | Head Writer                                                            | White Haven    | PA
 Kenneth Parcell | Page                                                                   | Stone Mountain | GA
 Jack Donaghy    | Vice President of East Coast Television and Microwave Oven Programming | Sadchester     | MA
(3 rows)
```

但是如果我们想查看视图的组成呢？好吧，我们可以像表格一样显示视图：

```sql
\d employee_hometowns
```
```
                  View "public.employee_hometowns"
  Column   |         Type          | Collation | Nullable | Default
-----------+-----------------------+-----------+----------+---------
 full_name | text                  |           |          |
 title     | character varying     |           |          |
 hometown  | character varying(80) |           |          |
 state     | character varying(2)  |           |          |
```

但这只是故事的一部分。它只显示表返回的内容，而不是它是如何生成的。PostgreSQL 9.6 引入了一个新命令来解决这个问题，`\sv`（`show view` 的缩写），让我们试试看。

```sql
\sv employee_hometowns
```
```
CREATE OR REPLACE VIEW public.employee_hometowns AS
 SELECT (employees.first_name::text || ' '::text) || employees.last_name::text AS full_name,
    employees.title,
    employees.hometown,
    cities.state
   FROM employees,
    cities
  WHERE employees.hometown::text = cities.name::text
```

这就是我们的视图的定义；相当酷啊！在任何视图上查看它是由什么组成的，而不是浏览迁移和脚本文件。这个 API 接口旨在扩展现有的 `\sf`（`show function` 的缩写）功能。例如，我们可以通过 `\sf` 查看 `now()` 函数的定义。

```sql
\sf now
```
```
CREATE OR REPLACE FUNCTION pg_catalog.now()
 RETURNS timestamp with time zone
 LANGUAGE internal
 STABLE PARALLEL SAFE STRICT
AS $function$now$function$
```

太好了，现在我们可以看到我们的视图，并了解它们的构成。但是如果我们也想编辑我们的视图呢？PostgreSQL 9.6 引入了视图编辑的命令。将 `s` 替换为 `e`，我们就可以编辑视图了。

```
\ev employee_hometowns
```
```
  1 CREATE OR REPLACE VIEW public.employee_hometowns AS
  2  SELECT (employees.first_name::text || ' '::text) || employees.last_name::text AS full_name,
  3     employees.title,
  4     employees.hometown,
  5     cities.state
  6    FROM employees,
  7     cities
  8   WHERE employees.hometown::text = cities.name::text
```

这将在您的默认文本编辑器中打开一个编辑缓冲区。编辑很有趣，并且有一些明确的边缘情况。您当然可以添加的一件事是列。让我们这样做，看看结果。

```
  1 CREATE OR REPLACE VIEW public.employee_hometowns AS
  2  SELECT (employees.first_name::text || ' '::text) || employees.last_name::text AS full_name,
  3     employees.title,
  4     employees.hometown,
  5     cities.state,
  6     cities.country
  7    FROM employees,
  8     cities
  9   WHERE employees.hometown::text = cities.name::text
```
```sql
SELECT * FROM employee_hometowns;
```
```
    full_name    |                                 title                                  |    hometown    | state | country
-----------------+------------------------------------------------------------------------+----------------+-------+---------
 Liz Lemon       | Head Writer                                                            | White Haven    | PA    | US
 Kenneth Parcell | Page                                                                   | Stone Mountain | GA    | US
 Jack Donaghy    | Vice President of East Coast Television and Microwave Oven Programming | Sadchester     | MA    | US
(3 rows)
```

现在我们可以看到我们的视图有一个 `country` 列。但是更改视图名称是有问题的，因为如果我们查看之前命令的 SQL 输出，它会根据其名称创建或替换视图。

```sql
\ev employee_hometowns
```
```
  1 CREATE OR REPLACE VIEW public.employee_hometowns_with_country AS
  2  SELECT (employees.first_name::text || ' '::text) || employees.last_name::text AS full_name,
  3     employees.title,
  4     employees.hometown,
  5     cities.state,
  6     cities.country
  7    FROM employees,
  8     cities
  9   WHERE employees.hometown::text = cities.name::text
```

```sql
\d employee_hometowns
```
```
                  View "public.employee_hometowns"
  Column   |         Type          | Collation | Nullable | Default
-----------+-----------------------+-----------+----------+---------
 full_name | text                  |           |          |
 title     | character varying     |           |          |
 hometown  | character varying(80) |           |          |
 state     | character varying(2)  |           |          |
 country   | character varying(2)  |           |          |
```
```sql
\d employee_hometowns_with_country
```
```
           View "public.employee_hometowns_with_country"
  Column   |         Type          | Collation | Nullable | Default
-----------+-----------------------+-----------+----------+---------
 full_name | text                  |           |          |
 title     | character varying     |           |          |
 hometown  | character varying(80) |           |          |
 state     | character varying(2)  |           |          |
 country   | character varying(2)  |           |          |
```

最后，我们不能删除或重命名视图返回的列。例如：

```sql
\ev employee_hometowns
```
```
  1 CREATE OR REPLACE VIEW public.employee_hometowns AS
  2  SELECT (employees.first_name::text || ' '::text) || employees.last_name::text AS full_name,
  3     employees.title,
  4     employees.hometown,
  5     cities.state
  6    FROM employees,
  7     cities
  8   WHERE employees.hometown::text = cities.name::text
```
```
ERROR:  cannot drop columns from view
```

为什么不能这样做呢？让我们查阅 PostgreSQL 文档：

> The new query must generate the same columns that were generated by the existing view query (that is, the same column names in the same order and with the same data types), but it may add additional columns to the end of the list. The calculations giving rise to the output columns may be completely different.

新查询必须生成与现有视图查询相同的列（也就是说，相同的列名以及相同的顺序和相同的数据类型），但它可能会在列末尾添加其他列。产生输出列的计算可能完全不同。

在您必须重命名视图或其列的情况下，只需删除视图然后重新创建它。

## 演示示例

```sql
-- create employees table
CREATE TABLE employees (
  hometown varchar(80),
  first_name varchar,
  last_name varchar,
  title varchar
);

-- create cities table
CREATE TABLE cities (
  name varchar(80),
  state varchar(2),
  country varchar(2)
);

-- populate data
INSERT INTO employees VALUES
  ('White Haven', 'Liz', 'Lemon', 'Head Writer'),
  ('Stone Mountain', 'Kenneth', 'Parcell', 'Page'),
  ('Sadchester', 'Jack', 'Donaghy', 'Vice President of East Coast Television and Microwave Oven Programming');

INSERT INTO cities VALUES
  ('White Haven', 'PA', 'US'),
  ('Stone Mountain', 'GA', 'US'),
  ('Sadchester', 'MA', 'US');

-- create view
CREATE VIEW employee_hometowns AS
  SELECT
    (first_name || ' ' || last_name) AS full_name,
    title,
    hometown,
    state
  FROM
    employees, cities
  WHERE
    hometown = name;
```

## 参考

[1] https://www.pgcasts.com/episodes/12/intro-to-views

[2] https://www.postgresql.org/docs/9.3/static/sql-createview.html

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第二十二集 [New in Postgres 9.6: Displaying and Editing Views](https://www.pgcasts.com/episodes/new-in-postgres-9-6-displaying-and-editing-views)。
