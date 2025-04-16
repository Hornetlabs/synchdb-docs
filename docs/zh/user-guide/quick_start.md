---
weight:  30
---
# 快速入门指南

只要您拥有正确的异构数据库连接信息，就可以非常轻松地开始使用 SynchDB 将数据从异构数据库复制到 PostgreSQL。请确保您已完成[此处](https://docs.synchdb.com/user-guide/installation/) 中描述的安装步骤。

## **创建 SynchDB 扩展**

SynchDB 扩展需要 pgcrypto 来加密某些敏感的凭证数据。请确保在安装 SynchDB 之前已安装它。或者，您可以在 `CREATE EXTENSION` 中包含 `CASCADE` 子句来自动安装依赖项：

```sql
CREATE EXTENSION synchdb CASCADE;
```

## **创建连接器**

这可以通过实用程序 SQL 函数 `synchdb_add_conninfo()` 来完成。

synchdb_add_conninfo 接受以下参数：

|        参数        | 描述 |
|-------------------- |-|
| name                  | 表示此连接器信息的唯一标识符 |
| hostname              | 异构数据库的 IP 地址或主机名 |
| port                  | 连接异构数据库的端口号 |
| username              | 用于异构数据库身份验证的用户名 |
| password              | 用于验证用户名的密码 |
| source database       | 这是我们要从中复制更改的异构数据库中的源数据库名称 |
| destination database  |（已弃用）始终默认使用与安装 synchDB 相同的数据库 |
| table                 | (可选) - 以 `[database].[table]` 或 `[database].[schema].[table]` 的形式表示，必须存在于异构数据库中，这样引擎将只复制指定的表。如果留空，则复制所有表 |
| connector             | 要使用的连接器类型（MySQL、Oracle、SQLServer 等）|

示例：

1. 创建一个名为 `mysqlconn` 的 MySQL 连接器，使用规则文件 `myrule.json` 从 MySQL 中的源数据库 `inventory` 复制到 PostgreSQL 中的目标数据库 `postgres`：
```sql
SELECT synchdb_add_conninfo(
    'mysqlconn',
    '127.0.0.1',
    3306,
    'mysqluser',
    'mysqlpwd',
    'inventory',
    'postgres',
    '',
    'mysql');
```

2. 创建一个名为 `mysqlconn2` 的 MySQL 连接器，使用默认转换规则从源数据库 `inventory` 复制到 PostgreSQL 中的目标数据库 `postgres`：
```sql
SELECT synchdb_add_conninfo(
    'mysqlconn2', '127.0.0.1', 3306, 'mysqluser', 
    'mysqlpwd', 'inventory', 'postgres', 
    '', 'mysql');
```

3. 创建一个名为 'sqlserverconn' 的 SQLServer 连接器，使用默认转换规则从源数据库 'testDB' 复制到 PostgreSQL 中的目标数据库 'postgres'：
```sql
SELECT 
  synchdb_add_conninfo(
    'sqlserverconn', '127.0.0.1', 1433, 
    'sa', 'Password!', 'testDB', 'postgres', 
    '', 'sqlserver');
```

4. 创建一个名为 `mysqlconn3` 的 MySQL 连接器，使用规则文件 `myrule2.json` 从源数据库 `inventory` 的 `orders` 和 `customers` 表复制到 PostgreSQL 中的目标数据库 `postgres`：
```sql
SELECT 
  synchdb_add_conninfo(
    'mysqlconn3', '127.0.0.1', 3306, 'mysqluser', 
    'mysqlpwd', 'inventory', 'postgres', 
    'inventory.orders,inventory.customers', 
    'mysql');
```

## **检查已创建的连接信息**

所有连接信息都创建在表 `synchdb_conninfo` 中。我们可以查看其内容并根据需要进行修改。请注意，用户凭证的密码是由 pgcrypto 使用仅 synchdb 知道的密钥加密的。因此，请不要修改密码字段，否则如果被篡改可能会解密错误。以下是输出示例：

```sql
postgres=# \x
Expanded display is on.

postgres=# select * from synchdb_conninfo;
-[ RECORD 1 ]-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
name     | sqlserverconn
isactive | t
data     | {"pwd": "\\xc30d0407030245ca4a983b6304c079d23a0191c6dabc1683e4f66fc538db65b9ab2788257762438961f8201e6bcefafa60460fbf441e55d844e7f27b31745f04e7251c0123a159540676c4", "port": 1433, "user": "sa", "dstdb": "postgres", "srcdb": "testDB", "table": "null", "hostname": "192.168.1.86", "connector": "sqlserver"}
-[ RECORD 2 ]-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
name     | mysqlconn
isactive | t
data     | {"pwd": "\\xc30d04070302986aff858065e96b62d23901b418a1f0bfdf874ea9143ec096cd648a1588090ee840de58fb6ba5a04c6430d8fe7f7d466b70a930597d48b8d31e736e77032cb34c86354e", "port": 3306, "user": "mysqluser", "dstdb": "postgres", "srcdb": "inventory", "table": "null", "hostname": "192.168.1.86", "connector": "mysql"}
-[ RECORD 3 ]-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
name     | oracleconn
isactive | t
data     | {"pwd": "\\xc30d04070302e3baf1293d0d553066d234014f6fc52e6eea425884b1f65f1955bf504b85062dfe538ca2e22bfd6db9916662406fc45a3a530b7bf43ce4cfaa2b049a1c9af8", "port": 1528, "user": "c##dbzuser", "dstdb": "postgres", "srcdb": "FREE", "table": "null", "hostname": "192.168.1.86", "connector": "oracle"}

```

## **启动连接器**

使用函数 `synchdb_start_engine_bgw()` 启动连接器工作进程。它接受一个参数，即上面创建的连接名称。此命令将生成一个新的后台工作进程，使用指定的配置连接到异构数据库。

例如，以下命令将在 PostgreSQL 中生成 2 个后台工作进程，一个从 MySQL 数据库复制，另一个从 SQL Server 复制：

```sql
select synchdb_start_engine_bgw('mysqlconn');
select synchdb_start_engine_bgw('sqlserverconn');
select synchdb_start_engine_bgw('oracleconn');
```

## **检查连接器运行状态**

使用“synchdb_state_view()”检查所有连接器的运行状态。

以下是输出示例：
``` SQL
postgres=# select * from synchdb_state_view;
     name      | connector_type |  pid   |        stage        |  state  |   err    |                                           last_dbz_offset
---------------+----------------+--------+---------------------+---------+----------+------------------------------------------------------------------------------------------------------
 sqlserverconn | sqlserver      | 579820 | change data capture | polling | no error | {"commit_lsn":"0000006a:00006608:0003","snapshot":true,"snapshot_completed":false}
 mysqlconn     | mysql          | 579845 | change data capture | polling | no error | {"ts_sec":1741301103,"file":"mysql-bin.000009","pos":574318212,"row":1,"server_id":223344,"event":2}
 oracleconn    | oracle         | 580053 | change data capture | polling | no error | offset file not flushed yet
(3 rows)

```

列详情：

| 字段            | 描述 |
|-|-|
| name   | 由 `synchdb_add_conninfo()` 创建的关联连接器信息名称 |
| connector_type       | 连接器类型（mysql、oracle、sqlserver 等）|
| pid             | 连接器工作进程的 PID |
| stage           | 连接器工作阶段 |
| state           | 连接器的状态。可能的状态有：<br><ul><li>stopped - 连接器未运行</li><li>initializing - 连接器正在初始化</li><li>paused - 连接器已暂停</li><li>syncing - 连接器正在定期轮询更改事件</li><li>parsing - 连接器正在解析收到的更改事件</li><li>converting - 连接器正在将更改事件转换为 PostgreSQL 表示</li><li>executing - 连接器正在将转换后的更改事件应用到 PostgreSQL</li><li>updating offset - 连接器正在向 Debezium 偏移量管理写入新的偏移量值</li><li>restarting - 连接器正在重启</li><li>dumping memory - 连接器正在输出 JVM 内存使用信息到 log 文件</li><li>unknown</li></ul> |
| err             | 工作进程遇到的最后一个错误消息，该错误可能导致其退出。此错误可能源自 PostgreSQL 处理更改时，或源自 Debezium 运行引擎从异构数据库访问数据时 |
| last_dbz_offset | synchdb 捕获的最后一个 Debezium 偏移量。请注意，这可能不反映连接器引擎的当前和实时偏移量值。相反，这显示为一个检查点，如果需要，我们可以从这个偏移量点重新启动 |

## **检查连接器运行统计信息**

使用 `synchdb_stats_view()` 视图检查所有连接器的统计信息。这些统计信息记录了连接器迄今为止处理过的各种类型变更事件的累积测量值。目前，这些统计值存储在共享内存中，并未持久化到磁盘。持久化统计数据是近期计划添加的功能。统计信息会在每次批次成功完成后更新，其中包含该批次中第一个和最后一个变更事件的多个时间戳。通过查看这些时间戳，我们可以粗略地了解完成批次处理所需的时间，以及数据生成和由 Debezium 和 PostgreQSL 处理之间的延迟。

以下是输出示例：
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
| name | 由 `synchdb_add_conninfo()` 创建的关联连接器信息名称|
| ddls | 完成的 DDL 操作数 |
| dmls | 完成的 DML 操作数 |
| reads | 初始快照阶段完成的读取事件数 |
| creates | CDC 阶段完成的创建事件数 |
| updates | CDC 阶段完成的更新事件数 |
| delets | CDC 阶段完成的删除事件数 |
| bad_events |忽略的不良事件数（例如空事件、不支持的 DDL 事件等）|
| total_events | 处理的事件总数（包括 bad_events）|
| batches_done | 完成的批次数 |
| avg_batch_size | 平均批次大小（total_events / batches_done）|
| first_src_ts | 外部数据库生成最后一个批次的第一个事件的时间戳（纳秒）|
| first_dbz_ts | Debezium 引擎处理最后一个批次的第一个事件的时间戳（纳秒）|
| first_pg_ts | 最后一个批次的第一个事件应用到 PostgreSQL 的时间戳（纳秒）|
| last_src_ts | 外部数据库生成最后一个批次的最后一个事件的时间戳（纳秒）|
| last_dbz_ts | Debezium 引擎处理最后一个批次的最后一个事件的时间戳（纳秒）|
| last_pg_ts | 最后一个批次的最后一个事件应用到 PostgreSQL 的时间戳（纳秒）|

## **停止连接器**

使用 SQL 函数 `synchdb_stop_engine_bgw()` 停止正在运行或暂停的连接器工作进程。此函数以 `conninfo_name` 作为其唯一参数，可以从 `synchdb_get_state()` 视图的输出中找到。

例如：
```sql
select synchdb_stop_engine_bgw('mysqlconn');
```

`synchdb_stop_engine_bgw()` 函数还会将连接信息标记为 `inactive`，这可以防止此工作进程在服务器重启时自动重新启动。

## **移除连接器**
使用 `synchdb_del_conninfo()` SQL 函数从 SynchDB 中移除连接器。这将清除该连接器上的[元数据文件](https://docs.synchdb.com/zh/architecture/metadata_files/)以及所有[对象映射](https://docs.synchdb.com/zh/user-guide/object_mapping_rules/)。

例如：
```
select synchdb_del_conninfo('mysqlconn');
```