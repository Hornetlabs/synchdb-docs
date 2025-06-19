---
weight:  30
---
# 快速入门指南

只要您拥有正确的异构数据库连接信息，就可以非常轻松地开始使用 SynchDB 将数据从异构数据库复制到 PostgreSQL。请确保您已完成[此处](https://docs.synchdb.com/zh/user-guide/installation/) 中描述的安装步骤，并准备好[示例数据库](https://docs.synchdb.com/zh/user-guide/prepare_tests_env/)进行测试（如果您尚未完成）。

## **创建 SynchDB 扩展**

SynchDB 扩展需要 pgcrypto 来加密某些敏感的凭证数据。请确保在安装 SynchDB 之前已安装它。或者，您可以在 `CREATE EXTENSION` 中包含 `CASCADE` 子句来自动安装依赖项：

```sql
CREATE EXTENSION synchdb CASCADE;
```

## **创建连接器**

以下是为每种受支持的源数据库类型创建基本连接器的一些示例。

1. 创建一个名为 `mysqlconn` 的 MySQL 连接器，将 MySQL 中 `inventory` 下的所有表复制到 PostgreSQL 中的目标数据库 `postgres`：
```sql
SELECT synchdb_add_conninfo(
'mysqlconn2', '127.0.0.1', 3306, 'mysqluser',
'mysqlpwd', 'inventory', 'postgres',
'null', 'null', 'mysql');
```

2. 创建一个名为 `sqlserverconn` 的 SQLServer 连接器，将 `testDB` 下的所有表复制到 PostgreSQL 中的目标数据库 `postgres`：
```sql
SELECT
synchdb_add_conninfo(
'sqlserverconn', '127.0.0.1', 1433,
'sa', 'Password!', 'testDB', 'postgres',
'null', 'null', 'sqlserver');
```

3. 创建一个名为 `oracleconn` 的 Oracle 连接器，将 `FREE` 下的所有表复制到 PostgreSQL 中的目标数据库 `postgres`：
```sql
SELECT
synchdb_add_conninfo(
'oracleconn', '127.0.0.1', 1521,
'c##dbzuser', 'dbz', 'FREE', 'postgres',
'null', 'null', 'oracle');
```

创建成功后，`synchdb_conninfo` 表下将有一条记录。有关创建连接器的更多详细信息，请参阅[此处](https://docs.synchdb.com/zh/user-guide/create_a_connector/)

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
如果连接器启动过程中出现错误，可能是由于连接性、用户凭据不正确或源数据库设置错误，则应捕获错误消息并在连接器状态输出中显示为 “err”。

有关运行状态的更多信息，请参见[此处](https://docs.synchdb.com/zh/monitoring/state_view/)，以及运行统计信息，请参见[此处](https://docs.synchdb.com/zh/monitoring/stats_view/)。

## 检查表和数据
默认情况下，源表将被复制到当前 postgreSQL 数据库下与源数据库同名的新架构中。我们可以更新搜索路径以查看不同架构中的新表。

例如：
```sql
postgres=# set search_path=public,free,inventory;
SET
postgres=# \d
                 List of relations
  Schema   |        Name        |   Type   | Owner
-----------+--------------------+----------+--------
 free      | orders             | table    | ubuntu
 inventory | addresses          | table    | ubuntu
 inventory | addresses_id_seq   | sequence | ubuntu
 inventory | customers          | table    | ubuntu
 inventory | customers_id_seq   | sequence | ubuntu
 inventory | geom               | table    | ubuntu
 inventory | geom_id_seq        | sequence | ubuntu
 inventory | perf_test_1        | table    | ubuntu
 inventory | products           | table    | ubuntu
 inventory | products_id_seq    | sequence | ubuntu
 inventory | products_on_hand   | table    | ubuntu
 public    | synchdb_att_view   | view     | ubuntu
 public    | synchdb_attribute  | table    | ubuntu
 public    | synchdb_conninfo   | table    | ubuntu
 public    | synchdb_objmap     | table    | ubuntu
 public    | synchdb_state_view | view     | ubuntu
 public    | synchdb_stats_view | view     | ubuntu
(17 rows)
```

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
```sql
select synchdb_del_conninfo('mysqlconn');
```