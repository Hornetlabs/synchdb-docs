# 实用函数列表


## synchdb_add_conninfo

用于创建新的连接器信息：

*  `name` - 代表此连接信息的唯一标识符

*  `hostname` - 异构数据库的 IP 地址

*  `port` - 要连接的端口号

*  `username` - 用于连接的用户名

*  `password` - 验证用户名的密码

*  `source database`（可选）- 要从中复制更改的数据库名称。例如，MySQL 中存在的数据库。如果为空，则复制 MySQL 中的所有数据库。

*  `destination database` - 要应用更改的数据库 - 例如，PostgreSQL 中存在的数据库。必须是存在的有效数据库。

*  `table`（可选）- 以 `[database].[table]` 的形式表示，必须存在于 MySQL 中，这样引擎只会复制指定的表。如果为空，则复制所有表。

*  `connector` - 要使用的连接器（MySQL、Oracle 或 SQLServer）。目前仅支持 MySQL 和 SQLServer。

*  `rule file` - 放置在 $PGDATA 下的 JSON 格式规则文件，该连接器将应用于其默认数据类型转换规则。详情见下文。

示例：
```
SELECT synchdb_add_conninfo('mysqlconn','127.0.0.1',3306,'mysqluser', 'mysqlpwd', 'inventory', 'postgres', '', 'mysql', 'myrule.json');
```

## synchdb_start_engine_bgw
用于启动连接器：
* `name` - 要启动的连接器名称

```
SELECT synchdb_start_engine_bgw('mysqlconn');
```

## synchdb_pause_engine
用于暂停正在运行的连接器：
* `name` - 要暂停的连接器名称
```
SELECT synchdb_pause_engine_bgw('mysqlconn');
```

## synchdb_resume_engine
用于恢复已暂停的连接器：
* `name` - 要恢复的连接器名称
```
SELECT synchdb_resume_engine('mysqlconn');
```

## synchdb_stop_engine_bgw
用于停止已暂停或正在运行的连接器：
* `name` - 要停止的连接器名称

```
SELECT synchdb_stop_engine('mysqlconn');
```

## synchdb_state_view
用于检查所有正在运行的连接器及其状态：
```
SELECT * FROM synchdb_state_view();
```

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

## synchdb_set_offset
用于设置自定义起始偏移量值：
* `name` - 要设置新偏移量的连接器名称
* `offset` - 要设置的偏移量值

```
SELECT synchdb_set_offset('mysqlconn', '{"ts_sec":1725644339,"file":"mysql-bin.000004","pos":138466,"row":1,"server_id":223344,"event":2}');
```