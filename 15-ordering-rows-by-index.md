# 按目标列编号排序

在这一集中，我们将看看在 `SELECT` 语句中对行进行排序时使用的便捷快捷方式。

假设我们有一个 `users` 表：

```sql
CREATE TABLE users (
  id serial primary key,
  first varchar not null,
  last varchar not null
);
```

我们在该表中有少量的用户，他们有名字和姓氏如下所示：

```sql
INSERT INTO users (first, last) VALUES
  ('Hank', 'Hooper'),
  ('Kathy', 'Geiss'),
  ('Devon', 'Banks'),
  ('Don', 'Geiss'),
  ('Jack', 'Donaghy');
```

我们想查看系统中的所有用户，因此我们在表上运行一个 `SELECT` 命令以获取名字和姓氏：

```sql
SELECT first, last FROM users;
```
```
 first |  last
-------+---------
 Hank  | Hooper
 Kathy | Geiss
 Devon | Banks
 Don   | Geiss
 Jack  | Donaghy
(5 rows)
```

很好，但是 `SELECT` 语句按照插入的顺序为我们提供了用户。我们真正想做的是查看按姓氏和名字排序的用户列表。我们可以通过使用 `ORDER BY` 子句来做到这一点：

```sql
SELECT first, last
FROM users
ORDER BY last, first;
```
```
 first |  last
-------+---------
 Devon | Banks
 Jack  | Donaghy
 Don   | Geiss
 Kathy | Geiss
 Hank  | Hooper
(5 rows)
```

这可能是我们通常看到的 `ORDER BY` 子句使用的方式。它还可以更灵活一些。我们可以通过引用输出列的编号，而不是直接使用输出列的名称。

我们的第一个输出列是 `first`，所以它的编号是 1。我们的第二个输出列是 `last`，所以它的编号是 2。让我们使用输出列的编号来实现与前一个相同的选择语句。

```sql
SELECT first, last
FROM users
ORDER BY 2, 1;
```
```
 first |  last
-------+---------
 Devon | Banks
 Jack  | Donaghy
 Don   | Geiss
 Kathy | Geiss
 Hank  | Hooper
(5 rows)
```

正如您所料，这些排序的默认值是升序的。我们可以将它们更改为降序，就像我们对任何其他 ORDER BY 子句所做的那样：

```sql
SELECT first, last
FROM users
ORDER BY 2 desc, 1 desc;
```

在这些示例中，我们并没有节省太多。然而，当我们在许多表之间构建复杂的语句时，使用编号被证明是非常有用的捷径。

## 译者著

[1] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第十五集 [Ordering Rows By Index](https://www.pgcasts.com/episodes/ordering-rows-by-index)。
