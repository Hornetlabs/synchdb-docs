# 变更日志
本项目的所有重要变更都将记录在此文件中。

本文格式基于 [Keep a Changelog](http://keepachangelog.com/)，
且本项目遵循 [语义化版本](http://semver.org/)。


## [SynchDB 1.0](https://github.com/Hornetlabs/synchdb/releases/tag/v1.0) - 2024-12-18

此版本侧重于 v1.0 Beta1 版本之后的错误修复和性能增强，使其在中高数据负载下更易于使用。更多与 Debezium 调优相关的参数也已作为 PostgreSQL GUC 公开，允许用户使用不同的参数进行测试。

### 添加
* 在 DML 解析阶段添加了数据缓存，以防止频繁访问 PostgreSQL catalog 以获取表的元组描述符结构。
* 添加 `synchdb_start_engine_bgw(name, mode)` 的变体，它接受第二个参数来指示启动连接器时使用的自定义快照模式。[此处](../user-guide/utility_functions)有更多详细信息。
* 添加多个可以调整 Debezium Runner 性能的 GUC。完整列表请参阅[此处](../user-guide/configuration.md)。
* 添加一个调试 SQL 函数 `synchdb_log_jvm_meminfo(name)`，该函数使指定的连接器在 PostgreSQL 日志文件中输出当前 JVM 堆内存使用情况摘要。
* 添加一个新视图 `synchdb_stats_view`，用于打印所有连接器的统计信息。更多详细信息请参阅[此处](../user-guide/utility_functions)。
* 增加了新的SQL函数 `synchdb_reset_stats(name)` 来清除指定连接器的统计信息。更多详细信息请参阅[此处](../user-guide/utility_functions)。
* 添加一个测试数据创建脚本，用于快速生成 MySQL 数据库类型的测试表和数据。

### 变更
* synchdb_state_view()：添加了一个名为 `stage` 的新字段，用于标识 connector 的当前阶段（值可以是 `Initial Snapshot` 或 `Change Data Capture`）。
* synchdb_state_view()：仅显示有效连接器的状态。
* 删除了在发生错误时向 Debezium Runner 发送 “部分批处理完成” 通知，因为批处理现在由一个 PostgreSQL 事务处理，且不允许部分完成。
* 现在可以在 [规则文件](../user-guide/transform_rule_file) 中指定 connector 所需的 SSL 相关参数。
* 现在可以通过 GUC 配置分配给运行 Debezium Runner 的 JVM 的最大堆内存。
* 现在可以配置连接器后台进程的最大数量，而不是硬编码的 30。

### 修复
* 在 Debezium runner 内添加节流控制，修复了 Debezium Runner 在数据量大时内存快速积累问题。
* 解决了 SynchDB 和 Debezium runner 组件中的大部分内存泄漏问题。
* 更正了 SynchDB 中 memory context 的使用，以便在每次更改事件处理结束时正确释放内存。
* 通过在单个 PostgreSQL 事务中处理一批改动，显著提高了 SynchDB 的处理速度。
* 将 SQLServer 的 `char` 类型的默认数据类型大小映射从 0 更正为 -1。
* 解决了使用 SPI 进行 DML 处理期间内存使用率过高的问题。

## [SynchDB 1.0 Beta1](https://github.com/Hornetlabs/synchdb/releases/tag/v1.0_beta1) - 2024-10-23

第一版 SynchDB 软件发布，为从异构数据库到 PostgreSQL 的无缝复制奠定了坚实的基础

### 新增
* 从异构数据库进行逻辑复制：（MySQL 和 SQLServer）。
* [DDL 复制](../user-guide/ddl_replication) (CREATE TABLE、DROP TABLE、ALTER TABLE ADD COLUMN、ALTER TABLE DROP COLUMN、ALTER TABLE ALTER COLUMN)
* DML 复制（INSERT、UPDATE、DELETE）。
* 最多 30 个并发连接器工作器。
* PostgreSQL 启动时 [自动连接器启动器](../user-guide/connector_auto_launcher)。
* 全局连接器状态和最后错误消息视图。
* [选择性数据库和表复制](../user-guide/selective_table_sync)。
* 批量更改事件。
* 连接器以不同的快照模式重新启动。
* [偏移管理接口](../user-guide/set_offset) 用于选择自定义复制恢复点。
* 支持的异构数据库的默认数据类型和对象名称转换规则。
* [JSON 规则文件](../user-guide/transform_rule_file) 用于定义自定义：（数据类型、列名、表名和数据表达式转换规则）。
* 2 种数据应用模式（SPI、HeapAM API）
* 新增[实用函数](../user-guide/utility_functions) 用于执行连接器操作：（启动、停止、暂停、恢复）。

### 变更

### 修复