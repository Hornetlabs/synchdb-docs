# 批量变更处理

## 概述
SynchDB 以 `synchdb.naptime` 毫秒（默认 500）的周期从 Debezium 运行引擎定期获取一批变更请求。这批变更请求随后由 SynchDB 处理。如果批次中的所有变更请求都已成功处理（解析、转换并应用到 PostgreSQL），SynchDB 将通知 Debezium 运行引擎该批次已完成。这向 Debezium 运行器发出信号，提交直到最后一条成功完成的变更记录的偏移量。通过这种机制，SynchDB 能够跟踪每条变更记录，并指示 Debezium 运行器不要获取之前已处理的旧变更，或不要发送重复的变更记录。

## 成功时的批处理
如果批次中的所有变更请求都已成功处理，SynchDB 只需向 Debezium 运行引擎发送一条消息，将该批次标记为已处理和完成，这会导致偏移量被提交并最终刷新到磁盘。

![img](https://www.highgo.ca/wp-content/uploads/2024/10/synchdb-Page-4.drawio.png)

## 部分成功时的批处理
如果由于内部 PostgreSQL 错误（如重复键违规）导致其中一个变更请求处理失败，SynchDB 仍会通知 Debezium 运行引擎关于部分完成的批次。这个通知消息包含一个指示器，表明最后一条成功处理的记录。Debezium 随后会将所有已完成的记录标记为已处理，同时保持未处理和失败的记录原样。这样，当 SynchDB 和 Debezium 运行器重新启动时，它们将从最后一条失败的变更记录恢复，并且只有在故障清除后才会继续处理。

![img](https://www.highgo.ca/wp-content/uploads/2024/10/synchdb-Page-5.drawio.png)