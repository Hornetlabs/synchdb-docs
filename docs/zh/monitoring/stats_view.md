# 连接器统计信息

## **检查连接器运行统计信息**

使用 `synchdb_stats_view()` 视图检查所有连接器的统计信息。这些统计信息记录了连接器迄今为止处理的不同类型变更事件的累积测量值。目前，这些统计值存储在共享内存中，而不是持久化到磁盘。持久化统计数据是近期将要添加的功能。统计信息会在每次批次成功完成后更新，其中包含该批次中第一个和最后一个变更事件的多个时间戳。通过查看这些时间戳，我们可以粗略地估算出处理完该批次所需的时间，以及数据生成和由 Debezium 和 PostgreQSL 处理之间的延迟。

请参阅以下示例输出：
```sql
postgres=# select * from synchdb_stats_view;
    name    | ddls | dmls | reads | creates | updates | deletes | bad_events | total_events | batches_done | avg_batch_size | first_src_ts  | first_dbz_ts  |  first_pg_ts  |  last_src_ts  |  last_dbz_ts  |  last_pg_ts
------------+------+------+-------+---------+---------+---------+------------+--------------+--------------+----------------+---------------+---------------+---------------+---------------+---------------+---------------
 oracleconn |    1 |    1 |     0 |       1 |       0 |       0 |          0 |            2 |            1 |              2 | 1744398189000 | 1744398230893 | 1744398231243 | 1744398198000 | 1744398230950 | 1744398231244
(1 row)
```
列详情：

| 字段 | 描述 |
|-|-|
| 名称 | 由 `synchdb_add_conninfo()` 创建的关联连接器信息名称 |
| ddls | 已完成的 DDL 操作数 |
| dmls | 已完成的 DML 操作数 |
| reads | 初始快照阶段完成的 READ 事件数 |
| create | CDC 阶段完成的 CREATES 事件数 |
| updates | CDC 阶段完成的 UPDATES 事件数 |
| deletes | CDC 阶段完成的 DELETES 事件数 |
| bad_events | 忽略的不良事件数（例如空事件、不支持的 DDL 事件等）|
| total_events | 已处理的事件总数（包括 bad_events）|
| batches_done | 已完成的批次数 |
| avg_batch_size | 平均批次大小（total_events / batches_done）|
| first_src_ts | 外部数据库生成最后一个批次的第一个事件的时间戳（纳秒）|
| first_dbz_ts | Debezium 引擎处理最后一个批次的第一个事件的时间戳（纳秒）|
| first_pg_ts | 最后一个批次的第一个事件应用到 PostgreSQL 的时间戳（纳秒）|
| last_src_ts | 外部数据库生成最后一个批次的最后一个事件的时间戳（纳秒）|
| last_dbz_ts | Debezium 引擎处理最后一个批次的最后一个事件的时间戳（纳秒）|
| last_pg_ts | 最后一个批次的最后一个事件应用到 PostgreSQL 的时间戳（纳秒）|