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
| stage | 连接器工作进程的阶段 |
| state | 连接器的状态。可能的状态有：<br><br><ul><li>已停止 - 连接器未运行</li><li>正在初始化 - 连接器正在初始化</li><li>已暂停 - 连接器已暂停</li><li>正在同步 - 连接器正在定期轮询变更事件</li><li>正在解析（连接器正在解析收到的变更事件）</li><li>正在转换 - 连接器正在将变更事件转换为 PostgreSQL 表示</li><li>正在执行 - 连接器正在将转换后的变更事件应用到 PostgreSQL</li><li>正在更新偏移量 - 连接器正在将新的偏移量值写入 Debezium 偏移量管理</li><li>正在重启 - 连接器正在重启</li><li>正在转储内存 - 连接器正在将 JVM 内存摘要转储到日志文件中</li><li>未知</li></ul> |
| err | 工作进程遇到的最后一条可能导致其退出的错误消息。此错误可能源于 PostgreSQL 处理更改时，也可能源于 Debezium 运行引擎访问异构数据库数据时。|
| last_dbz_offset | synchdb 捕获的最后一个 Debezium 偏移量。请注意，这可能无法反映连接器引擎的当前和实时偏移量值。相反，它显示为一个检查点，我们可以根据需要从此偏移点重新启动。|