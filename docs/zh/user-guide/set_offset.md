---
weight: 100
---
# 自定义起始偏移量值

起始偏移量值代表开始复制的点，类似于 PostgreSQL 的恢复 LSN。当 Debezium 运行引擎启动时，它将从这个偏移量值开始复制。将此偏移量值设置为较早的值将导致 Debezium 运行引擎从较早的记录开始复制，可能会复制重复的数据记录。在设置 Debezium 的起始偏移量值时，我们应该格外谨慎。

## **记录可设置的偏移量值**

在操作过程中，Debezium 运行引擎将生成新的偏移量并将其刷新到磁盘。最后刷新的偏移量可以通过 `synchdb_state_view()` 实用命令检索：
```
postgres=# select name, last_dbz_offset from synchdb_state_view;
     name      |                                           last_dbz_offset
---------------+------------------------------------------------------------------------------------------------------
 sqlserverconn | {"commit_lsn":"0000006a:00006608:0003","snapshot":true,"snapshot_completed":false}
 mysqlconn     | {"ts_sec":1741301103,"file":"mysql-bin.000009","pos":574318212,"row":1,"server_id":223344,"event":2}
 oracleconn    | offset file not flushed yet
(3 rows)

```

根据连接器类型的不同，这个偏移量值也不同。从上面的示例中，`mysql` 连接器的最后刷新偏移量是 `{"ts_sec":1741301103,"file":"mysql-bin.000009","pos":574318212,"row":1,"server_id":223344,"event":2}`，而 `sqlserver` 的最后刷新偏移量是 `{"commit_lsn":"0000006a:00006608:0003","snapshot":true,"snapshot_completed":false}`。

我们应该定期保存这些值，这样如果遇到问题，我们就知道过去可以设置的偏移量位置，以恢复复制操作。

## **暂停连接器**

在设置新的偏移量值之前，连接器必须处于 `paused`（暂停）状态。

使用 `synchdb_pause_engine()` SQL 函数暂停正在运行的连接器。这将停止 Debezium 运行引擎从异构数据库复制。当暂停时，可以使用 `synchdb_set_offset()` SQL 例程更改 Debezium 连接器的偏移量值，以从过去的特定点开始复制。它以 `conninfo_name` 作为参数，可以从 `synchdb_get_state()` 视图的输出中找到。

例如：
```
SELECT synchdb_pause_engine('mysqlconn');
```

## **设置新的偏移量**

使用 `synchdb_set_offset()` SQL 函数更改连接器工作进程的起始偏移量。只有当连接器处于 `paused` 状态时才能执行此操作。该函数接受两个参数，`conninfo_name` 和 `有效的偏移量字符串`，这两个参数都可以从 `synchdb_get_state()` 视图的输出中找到。

例如：
```
SELECT synchdb_set_offset('mysqlconn', '{"ts_sec":1741301103,"file":"mysql-bin.000009","pos":574318212,"row":1,"server_id":223344,"event":2}');
```

## **恢复连接器**

使用 `synchdb_resume_engine()` SQL 函数从暂停状态恢复 Debezium 操作。此函数以 `连接器名称` 作为其唯一参数，可以从 `synchdb_get_state()` 视图的输出中找到。恢复的 Debezium 运行引擎将从新设置的偏移量值开始复制。

例如：
```
SELECT synchdb_resume_engine('mysqlconn');
```