# 连接器运行状态

## 检查连接器运行状态
使用 `synchdb_state_view()` 检查所有连接器的运行状态。

请参阅以下示例输出：
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

| 字段 | 描述 |
|-|-|
| name | 由 `synchdb_add_conninfo()` 创建的关联连接器信息名称 |
| connector_type | 连接器类型（mysql、oracle、sqlserver 等）|
| pid | 连接器工作进程的 PID |
| stage | 连接器的阶段。见下文。|
| state | 连接器的状态。见下文。|
| err | 工作进程遇到的最后一个可能导致其退出的错误消息。此错误可能源于 PostgreSQL 处理更改时，也可能源于 Debezium 运行引擎访问异构数据库数据时。|
| last_dbz_offset | synchdb 捕获的最后一个 Debezium 偏移量。请注意，这可能无法反映连接器引擎的当前实时偏移量值。相反，它显示为一个检查点，我们可以根据需要从此偏移量点重新启动。|

**可能的状态**：

- 🔴 `stopped` - 非活动
- 🟡 `initializing` - 正在启动
- 🟠 `paused` - 暂时停止
- 🟢 `syncing` - 正在主动轮询
- 🔵 `parsing` - 正在处理事件
- 🟣 `converting` - 正在转换数据
- ⚪ `executing` - 正在应用更改
- 🟤 `updating offset` - 正在更新检查点
- 🟨 `restarting` - 正在重新初始化
- ⚪ `dumping memory` - JVM 正在准备将内存信息转储到日志文件
- ⚫ `unknown` - 不确定状态

**可能的状态**：

- `initial snapping` - 连接器正在执行初始快照（构建表结构以及可选的初始数据）
- `change data capture` - 连接器正在流式传输后续表更改 (CDC)
- `schema sync` - 连接器仅复制表架构（无数据）