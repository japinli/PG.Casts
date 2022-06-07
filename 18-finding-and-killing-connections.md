# 查找并终止连接

在本集中，Josh Branchaud 将向您展示如何在 PostgreSQL 中终止空闲连接。本周早些时候，我遇到了与项目数据库连接的问题。连接应该几乎立即关闭，但它却无所事事。这个细节并不重要。在各种情况下，您最终可能会与数据库建立不需要的连接。我们要关注的是如何追踪连接然后将其终止。

现在，有各种各样的工具和方法可以跟踪和杀死进程。不过，我们将要做的事情更有趣，因为这一切都将发生在 PSQL 会话中。

首先，让我们通过连接到我们的数据库来创建到我们的数据库的连接。

```shell
$ psql pgcasts
```

为了清楚地了解正在发生的事情，我将启动一个事务，创建一个包含一些行的表，然后从该表中选择所有行：

```sql
BEGIN;
CREATE TABLE franks_hats ( slogan varchar primary key );
INSERT INTO franks_hats VALUES
  ('Ninja Expert'), ('Done Deal'), ('Double Cheese'), ('Arcade Champ');

SELECT * FROM FRANKS_HATS;
```

现在我们的空闲连接已经就绪。让我们打开另一个到数据库的连接。

```shell
$ # open new tmux pane
$ psql pgcasts
```

正是从这个连接中，我们可以追踪并终止我们的空闲连接。除了我们通常认为 PostgreSQL 为我们做的所有事情之外，它还监视服务器进程活动。PostgreSQL 通过 `pg_stat_activity` 视图公开这些信息。被监控的进程之一是我们的空闲连接。通过正确的查询，我们可以找到该进程。

那么，这个视图暴露了哪些信息呢？

```sql
\d pg_stat_activity
```
```
                      View "pg_catalog.pg_stat_activity"
      Column      |           Type           | Collation | Nullable | Default
------------------+--------------------------+-----------+----------+---------
 datid            | oid                      |           |          |
 datname          | name                     |           |          |
 pid              | integer                  |           |          |
 leader_pid       | integer                  |           |          |
 usesysid         | oid                      |           |          |
 usename          | name                     |           |          |
 application_name | text                     |           |          |
 client_addr      | inet                     |           |          |
 client_hostname  | text                     |           |          |
 client_port      | integer                  |           |          |
 backend_start    | timestamp with time zone |           |          |
 xact_start       | timestamp with time zone |           |          |
 query_start      | timestamp with time zone |           |          |
 state_change     | timestamp with time zone |           |          |
 wait_event_type  | text                     |           |          |
 wait_event       | text                     |           |          |
 state            | text                     |           |          |
 backend_xid      | xid                      |           |          |
 backend_xmin     | xid                      |           |          |
 query_id         | bigint                   |           |          |
 query            | text                     |           |          |
 backend_type     | text                     |           |          |
```

对我来说最突出的是 `datname`，我们数据库的名称； `pid`，进程标识；和 `query`，这个连接上最近执行的查询。

让我们看看如果我们获取当前数据库的 `pid` 和 `query` 列会发现什么：

```sql
SELECT pid, query FROM pg_stat_activity WHERE datname = current_database();
```
```
   pid   |                                    query
---------+-----------------------------------------------------------------------------
 4160728 | SELECT pid, query FROM pg_stat_activity WHERE datname = current_database();
 4130309 | SELECT * FROM FRANKS_HATS;
(2 rows)
```

我们得到两个结果，它们很能说明问题。一个是我们当前的连接，另一个是我们的空闲连接。有了空闲进程的 `pid`，我们现在可以使用 `pg_terminate_backend(int)` 函数来终止连接。

```sql
SELECT pg_terminate_backend(4130309);
```
```
 pg_terminate_backend
----------------------
 t
(1 row)
```

执行结果为 `true` 时意味着连接成功被终止。然而，我们的另一个连接看起来并没有发生什么。让我们再试试从那个连接重新运行我们之前的查询。

```sql
SELECT * FROM franks_hats;
```
```
FATAL:  terminating connection due to administrator command
server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.
The connection to the server was lost. Attempting reset: Succeeded.
```

实际上，连接已终止。在这种情况下，psql 能够重新连接我们的会话。因为连接被终止了，所以任何事务都将被回滚并释放锁。

我们可以再次尝试这个查询，并看到我们之前的事务确实被回滚了：

```sql
SELECT * FROM franks_hats;
```
```
ERROR:  relation "franks_hats" does not exist
LINE 1: SELECT * FROM franks_hats;
                      ^
```

您可能已经注意到，`pg_stat_activity` 视图中包含更多信息，因此请随时进一步探索。另外，请记住 `pg_terminate_backend()` 和类似的函数需要系统管理员权限才能成功运行它们。查看参考中的链接以获取更多详细信息。

## 参考

[1] [pg_stat_activity 视图](https://www.postgresql.org/docs/current/static/monitoring-stats.html#PG-STAT-ACTIVITY-VIEW)

[2] [pg_terminate_backend(pid) 和其它系统管理函数](https://www.postgresql.org/docs/current/static/functions-admin.html)

## 译者著

[1] 本文中的输出基于 PostgreSQL 15devel，因此可能与原文中存在一定的差异。

[2] 本文翻译自 [PG Casts](https://www.pgcasts.com/) 的第十八集 [Finding and Killing Connections](https://www.pgcasts.com/episodes/finding-and-killing-connections)。
