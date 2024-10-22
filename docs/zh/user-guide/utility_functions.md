# 实用函数列表


## synchdb_add_conninfo

用于创建新的连接器信息：

|       参数        |                                                                 描述                                                                 |
|:-----------------:|:------------------------------------------------------------------------------------------------------------------------------------:|
| name              | 此连接器信息的唯一标识符                                                                                                              |
| hostname          | 异构数据库的 IP 地址或主机名                                                                                                         |
| port              | 连接到异构数据库的端口号                                                                                                              |
| username          | 用于验证异构数据库身份的用户名                                                                                                        |
| password          | 验证用户名的密码                                                                                                                     |
| source database   | 异构数据库中我们想要复制更改的源数据库名称                                                                                            |
| destination database | 这是要将更改应用到的 PostgreSQL 目标数据库名称。它必须是 PostgreSQL 中存在的有效数据库                                               |
| table             | （可选） - 表达形式为 `[database].[table]` 或 `[database].[schema].[table]`，必须在异构数据库中存在，只有指定的表会被复制。如果留空，则复制所有表 |
| connector         | 要使用的连接器类型（MySQL、Oracle、SQLServer 等）                                                                                       |
| rule file         | 放置在 $PGDATA 下的 JSON 格式规则文件，此连接器将应用其默认的数据类型转换规则。更多信息请参见 [这里](https://docs.synchdb.com/user-guide/transform_rule_file/) |

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

| 字段                | 描述                                                                                                                                                                                                                       |
|---------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| id                  | 连接器槽的唯一标识符                                                                                                                                                                                                      |
| connector           | 连接器类型（mysql、oracle、sqlserver 等）                                                                                                                                                                                 |
| conninfo_name       | 通过 `synchdb_add_conninfo()` 创建的关联连接器信息名称                                                                                                                                                                     |
| pid                 | 连接器工作进程的 PID                                                                                                                                                                                                      |
| state               | 连接器的状态。可能的状态包括：<br><br><ul><li>stopped - 连接器未运行</li><li>initializing - 连接器正在初始化</li><li>paused - 连接器已暂停</li><li>syncing - 连接器定期轮询更改事件</li><li>parsing - 连接器正在解析收到的更改事件</li><li>converting - 连接器正在将更改事件转换为 PostgreSQL 表示形式</li><li>executing - 连接器正在将转换后的更改事件应用到 PostgreSQL</li><li>updating offset - 连接器正在将新偏移量写入 Debezium 偏移管理</li><li>restarting - 连接器正在重启</li><li>unknown</li></ul> |
| err                 | 工作进程遇到的最后错误信息，导致其退出。此错误可能来自 PostgreSQL 在处理更改时发生，或者来自 Debezium 引擎在访问异构数据库时发生                                                     |
| last_dbz_offset     | 同步数据库捕获的最后一个 Debezium 偏移量。请注意，这可能无法反映连接器引擎的当前实时偏移量，而是显示为一个检查点，如果需要可以从此偏移点重新启动 |

## synchdb_set_offset
用于设置自定义起始偏移量值：
* `name` - 要设置新偏移量的连接器名称
* `offset` - 要设置的偏移量值

```
SELECT synchdb_set_offset('mysqlconn', '{"ts_sec":1725644339,"file":"mysql-bin.000004","pos":138466,"row":1,"server_id":223344,"event":2}');
```

## synchdb_restart_connector
用于以不同的快照模式重启连接器
* `name` - 要重启的连接器名称。必须已经在运行。
* `snapshot_mode` - 重启时使用的快照模式

此 SQL 函数可用于快速以 `initial`（默认快照模式）以外的其他快照模式重启连接器。我们可以参考下表使用不同的快照模式：

| **设置**       | **描述**                                                                                                                 |
|:--------------:|:-------------------------------------------------------------------------------------------------------------------------:|
| always         | 连接器每次启动时都会执行快照。快照包括捕获表的结构和数据。快照完成后，连接器开始流式传输后续数据库更改的事件记录。              |
| initial        | 如果尚未执行，连接器将执行数据库快照。快照完成后，连接器开始流式传输后续数据库更改的事件记录。                               |
| initial_only   | 连接器执行数据库快照。快照完成后，连接器停止，不会流式传输后续数据库更改的事件记录。                                           |
| no_data        | 连接器捕获所有相关表的结构，但不捕获它们包含的数据。                                                                         |
| never          | 当连接器启动时，它不会执行快照，而是立即开始流式传输后续数据库更改的事件记录。                                                 |
| recovery       | 设置此选项以恢复丢失或损坏的数据库模式历史记录。重启后，连接器运行快照以从源表重建主题。                                        |
| when_needed    | 连接器启动后，只有在检测到以下情况之一时才执行快照：<br><br><ul><li>无法检测到任何主题偏移</li><li>之前记录的偏移指定了服务器上不可用的日志位置</li></ul> |