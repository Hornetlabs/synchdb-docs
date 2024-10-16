---
title: 快速开始
---
# 快速入门指南

只要您有正确的异构数据库连接信息，使用 SynchDB 从异构数据库复制数据到 PostgreSQL 是非常简单的。

## 安装 SynchDB 扩展
SynchDB 扩展需要 pgcrypto 来加密某些敏感的凭证数据。请确保在安装 SynchDB 之前已安装它。或者，您可以在 `CREATE EXTENSION` 中包含 `CASCADE` 子句以自动安装依赖项：
```
CREATE EXTENSION synchdb CASCADE;
```

## 创建连接信息
这可以通过实用 SQL 函数 `synchdb_add_conninfo()` 完成。

synchdb_add_conninfo 接受以下参数：

* `name` - 代表此连接信息的唯一标识符
* `hostname` - 异构数据库的 IP 地址。
* `port` - 连接异构数据库的端口号。
* `username` - 要使用的用户名。
* `password` - 验证用户名的密码。
* `source database` - 这是我们要从中复制更改的异构数据库中的源数据库名称。
* `destination database` - 这是要应用更改的 PostgreSQL 中的目标数据库名称。它必须是 PostgreSQL 中存在的有效数据库。
* `table`（可选）- 以 `[database].[table]` 或 `[database].[schema].[table]` 的形式表示，必须存在于异构数据库中，引擎将只复制指定的表。如果留空，将复制所有表。请注意，未在此选择的表也将在 PostgreSQL 中复制其表结构，但只有选定表的数据会被复制。
* `connector` - 要使用的连接器类型（MySQL、Oracle、SQLServer 等）。
* `rule file` - 放置在 $PGDATA 下的 JSON 格式规则文件，该连接器将应用于其默认数据类型转换规则。

示例：

1. 创建一个名为 `mysqlconn` 的 MySQL 连接器，从 MySQL 的源数据库 `inventory` 复制到 PostgreSQL 的目标数据库 `postgres`，使用规则文件 `myrule.json`：
```
SELECT synchdb_add_conninfo('mysqlconn', '127.0.0.1', 3306, 'mysqluser', 'mysqlpwd', 'inventory', 'postgres', '', 'mysql', 'myrule.json');
```

2. 创建一个名为 `mysqlconn2` 的 MySQL 连接器，从源数据库 `inventory` 复制到 PostgreSQL 的目标数据库 `mysqldb2`，使用默认转换规则：
```
SELECT synchdb_add_conninfo('mysqlconn2', '127.0.0.1', 3306, 'mysqluser', 'mysqlpwd', 'inventory', 'mysqldb2', '', 'mysql', '');
```

3. 创建一个名为 'sqlserverconn' 的 SQLServer 连接器，从源数据库 'testDB' 复制到 PostgreSQL 的目标数据库 'sqlserverdb'，使用默认转换规则：
```
SELECT synchdb_add_conninfo('sqlserverconn', '127.0.0.1', 1433, 'sa', 'Password!', 'testDB', 'sqlserverdb', '', 'sqlserver', '');
```

4. 创建一个名为 `mysqlconn3` 的 MySQL 连接器，从源数据库 `inventory` 的 `orders` 和 `customers` 表复制到 PostgreSQL 的目标数据库 `mysqldb3`，使用规则文件 `myrule2.json`：
```
SELECT synchdb_add_conninfo('mysqlconn3', '127.0.0.1', 3306, 'mysqluser', 'mysqlpwd', 'inventory', 'mysqldb3', 'inventory.orders,inventory.customers', 'mysql', 'myrule2.json');
```
## 注意事项
* 可以创建多个连接到同一连接器类型（如 MySQL、SQLServer 等）的连接器。SynchDB 将生成单独的连接来获取变更数据。
* 避免创建两个连接到同一异构数据库、相同源数据库、表和相同目标数据库的连接器。这可能在两个连接器启动时造成冲突。将来，我们将添加列和表名映射功能，允许将源表映射到目标上的不同名称，以防止这种名称冲突。同时，请使用不同的目标数据库。
* 近期将支持用户定义的 X509 证书和私钥，用于与远程数据库的 TLS 连接。同时，请确保 TLS 设置设为可选。

## 检查创建的连接信息
所有连接信息都创建在 `synchdb_conninfo` 表中。我们可以自由查看其内容并根据需要进行修改。请注意，用户凭证的密码是由 pgcrypto 使用只有 synchdb 知道的密钥加密的。因此，请不要修改密码字段，否则如果被篡改，可能会被错误解密。以下是示例输出：
```
postgres=# \x
Expanded display is on.

postgres=# select * from synchdb_conninfo;
-[ RECORD 1 ]-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
name | mysqlconn
data | {"pwd": "\\xc30d040703024828cc4d982e47b07bd23901d03e40da5995d2a631fb89d49f748b87247aee94070f71ecacc4990c3e71cad9f68d57c440de42e35bcc78fd145feab03452e454284289db", "port": 3306, "user": "mysqluser", "dstdb": "postgres", "srcdb": "inventory", "table": "null", "hostname": "192.168.1.86", "connector": "mysql"i, "myrule.json"}
-[ RECORD 2 ]-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
name | sqlserverconn
data | {"pwd": "\\xc30d0407030231678e1bb0f8d3156ad23a010ca3a4b0ad35ed148f8181224885464cdcfcec42de9834878e2311b343cd184fde65e0051f75d6a12d5c91d0a0403549fe00e4219215eafe1b", "port": 1433, "user": "sa", "dstdb": "sqlserverdb", "srcdb": "testDB", "table": "null", "hostname": "192.168.1.86", "connector": "sqlserver", "null"}
```

## 启动连接器
使用 `synchdb_start_engine_bgw()` 函数启动连接器工作进程。它接受一个参数，即上面创建的连接名称。此命令将在 PostgreSQL 中生成一个新的后台工作进程，以使用指定的配置连接到异构数据库。

例如，以下命令将在 PostgreSQL 中生成 2 个后台工作进程，一个从 MySQL 数据库复制，另一个从 SQL Server 复制：
```
select synchdb_start_engine_bgw('mysqlconn');
select synchdb_start_engine_bgw('sqlserverconn');
```

## 检查连接器运行状态
使用 `synchdb_state_view()` 视图检查所有正在运行的连接器及其状态。目前，synchdb 最多可支持 30 个运行中的工作进程。

以下是示例输出：
```
postgres=# select * from synchdb_state_view;
 id | connector | conninfo_name  |  pid   |  state  |   err    |                                          last_dbz_offset
----+-----------+----------------+--------+---------+----------+---------------------------------------------------------------------------------------------------
  0 | mysql     | mysqlconn      | 461696 | syncing | no error | {"ts_sec":1725644339,"file":"mysql-bin.000004","pos":138466,"row":1,"server_id":223344,"event":2}
  1 | sqlserver | sqlserverconn  | 461739 | syncing | no error | {"event_serial_no":1,"commit_lsn":"00000100:00000c00:0003","change_lsn":"00000100:00000c00:0002"}
  3 | null      |                |     -1 | stopped | no error | no offset
  4 | null      |                |     -1 | stopped | no error | no offset
  4 | null      |                |     -1 | stopped | no error | no offset
  5 | null      |                |     -1 | stopped | no error | no offset
  6 | null      |                |     -1 | stopped | no error | no offset
  7 | null      |                |     -1 | stopped | no error | no offset
  8 | null      |                |     -1 | stopped | no error | no offset
  9 | null      |                |     -1 | stopped | no error | no offset
 10 | null      |                |     -1 | stopped | no error | no offset
 11 | null      |                |     -1 | stopped | no error | no offset
 12 | null      |                |     -1 | stopped | no error | no offset
 13 | null      |                |     -1 | stopped | no error | no offset
 14 | null      |                |     -1 | stopped | no error | no offset
 15 | null      |                |     -1 | stopped | no error | no offset
 16 | null      |                |     -1 | stopped | no error | no offset
 17 | null      |                |     -1 | stopped | no error | no offset
 18 | null      |                |     -1 | stopped | no error | no offset
 19 | null      |                |     -1 | stopped | no error | no offset
 20 | null      |                |     -1 | stopped | no error | no offset
 21 | null      |                |     -1 | stopped | no error | no offset
 22 | null      |                |     -1 | stopped | no error | no offset
 23 | null      |                |     -1 | stopped | no error | no offset
 24 | null      |                |     -1 | stopped | no error | no offset
 25 | null      |                |     -1 | stopped | no error | no offset
 26 | null      |                |     -1 | stopped | no error | no offset
 27 | null      |                |     -1 | stopped | no error | no offset
 28 | null      |                |     -1 | stopped | no error | no offset
 29 | null      |                |     -1 | stopped | no error | no offset
```


列详情：

* id：连接器槽的唯一标识符

* connector：连接器类型（mysql、oracle、sqlserver 等）

* conninfo_name：由 `synchdb_add_conninfo()` 创建的关联连接信息名称

* pid：连接器工作进程的 PID

* state：连接器的状态。可能的状态有：

* stopped（已停止）

* initializing（初始化中）

* paused（已暂停）

* syncing（同步中）

* parsing（解析中）

* converting（转换中）

* executing（执行中）

* updating offset（更新偏移量）

* unknown（未知）

* err：工作进程遇到的最后一个错误消息，可能导致其退出。这个错误可能源自 PostgreSQL 处理变更时，或源自 Debezium 运行引擎访问异构数据库时。

* last_dbz_offset：synchdb 捕获的最后一个 Debezium 偏移量。注意，这可能不反映连接器引擎的当前和实时偏移量值。相反，这显示为一个检查点，如果需要，我们可以从这个偏移点重新启动。

## 停止连接器
使用 `synchdb_stop_engine_bgw()` SQL 函数停止正在运行或暂停的连接器工作进程。此函数以 `conninfo_name` 作为其唯一参数，可以从 `synchdb_get_state()` 视图的输出中找到。

例如：
```
select synchdb_stop_engine_bgw('mysqlconn');
```

`synchdb_stop_engine_bgw()` 函数还将连接信息标记为 `inactive`，这可以防止这个工作进程在服务器重启时自动重新启动。有关更多详细信息，请参见下文。