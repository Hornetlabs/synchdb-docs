# 变更日志
本项目的所有重要变更都将记录在此文件中。

本文格式基于 [Keep a Changelog](http://keepachangelog.com/)，
且本项目遵循 [语义化版本](http://semver.org/)。

## [SynchDB 1.0 Beta1] - 2024-10-23

第一版 SynchDB 软件发布，为从异构数据库到 PostgreSQL 的无缝复制奠定了坚实的基础

### 新增
* 从异构数据库进行逻辑复制：（MySQL 和 SQLServer）。
* [DDL 复制](user-guide/ddl_replication) (CREATE TABLE、DROP TABLE、ALTER TABLE ADD COLUMN、ALTER TABLE DROP COLUMN、ALTER TABLE ALTER COLUMN)
* DML 复制（INSERT、UPDATE、DELETE）。
* 最多 30 个并发连接器工作器。
* PostgreSQL 启动时 [自动连接器启动器](user-guide/connector_auto_launcher)。
* 全局连接器状态和最后错误消息视图。
* [选择性数据库和表复制](user-guide/selective_table_sync)。
* 批量更改事件。
* 连接器以不同的快照模式重新启动。
* [偏移管理接口](user-guide/set_offset) 用于选择自定义复制恢复点。
* 支持的异构数据库的默认数据类型和对象名称转换规则。
* [JSON 规则文件](user-guide/transform_rule_file) 用于定义自定义：（数据类型、列名、表名和数据表达式转换规则）。
* 2 种数据应用模式（SPI、HeapAM API）
* 新增[实用函数](user-guide/utility_functions) 用于执行连接器操作：（启动、停止、暂停、恢复）。

### 变更

### 修复