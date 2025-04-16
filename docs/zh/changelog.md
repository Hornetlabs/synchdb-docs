# 变更日志
本项目的所有重要变更都将记录在此文件中。

本文格式基于 [Keep a Changelog](http://keepachangelog.com/)，
且本项目遵循 [语义化版本](http://semver.org/)。

## **[SynchDB 1.1](https://github.com/Hornetlabs/synchdb/releases/tag/v1.1) - 2025-04-17**

此版本引入了对 Oracle 连接器的支持，以及针对 Oracle 数据源量身定制的强大数据类型转换功能。SynchDB 的核心数据处理引擎得到了显著改进，能够更可靠地处理复杂多样的数据转换组合。

性能也在多个方面得到了提升。通过智能缓存消除了冗余迭代和目录查找，并通过采用更高效的 PostgreSQL JSON 解析器 API 显著提升了 JSON 解析性能。这些增强功能降低了开销，并有助于实现更快、更可扩展的复制。

SynchDB 现在完全支持 PostgreSQL 16, 17 和 IvorySQL 4.4，确保与最新的基于 PostgreSQL 的平台兼容。此外，通过集成对象映射管理实用程序，定义自定义转换规则的流程得到了改进，提供了一种更直观、更易于维护的转换逻辑配置和扩展方式。

### **新增**

* 新增 Oracle 连接器支持。
* 新增 PostgreSQL 16, 17 和 IvorySQL 4.4 的支持。
* 新增 Oracle 可变大小/规模数据类型 `NUMBER` 处理的支持。
* 新增 GUC `synchdb.error_handling_strategy` 来控制错误处理策略：可以是跳过、退出或重试。
* 新增 GUC `synchdb.dbz_log_level` 来控制 Debezium 运行引擎使用的 log4j 日志级别。
* 新增表 `synchdb_attribute`，用于存储 synchdb 当前捕获的所有远程表属性信息。
* 新增视图 `synchdb_att_view`，用于并排显示数据类型、表和列名映射的比较。
* 新增表 `synchdb_objmap`，用于存储不同的对象映射条目。
* 新增 SQL 函数 `synchdb_add_objamp()`，用于添加新的对象映射条目。该函数支持表名、列名、数据类型和转换表达式。
* 新增 SQL 函数 `synchdb_reload_objmap()`，用于强制连接器重新加载对象映射条目。
* 新增 `schemasync` 的新快照模式，在该模式下，连接器将从远程数据库获取模式信息，创建同步表并切换到暂停状态。
* 新增 `synchdb_add_extra_conninfo()` 函数，用于为连接器配置额外的 SSL 相关连接参数。
* 新增 `synchdb_del_extra_conninfo()` 函数，用于删除 `synchdb_add_extra_conninfo()` 创建的所有额外参数。
* 新增 `synchdb_del_conninfo()` 函数，用于删除现有连接器信息、所有相关的元数据和对象映射，并根据需要关闭连接器。
* 新增 `synchdb_del_objmap()` 函数，用于禁用和移除对象映射条目。

### **已更改**

* 处理 ALTER TABLE 变更事件时，synchdb 仅在表本身没有主键时添加主键。
* 移除了在 `synchdb_add_conninfo()` 中配置的目标数据库名。 默认使用 synchdb 安装的所在数据库名。
* 在每个 DDL 变更事件处理结束时，synchdb 现在都会对新的 `synchdb_attribute` 表进行更新。
* 将所有数据类型映射条目规范化为使用小写字母。
* 当启动或重新加载时，如果配置的对象映射与当前值不同，连接器将更正表名、列名和数据类型。
* `rulefile` 已被 `synchdb_add_objmap()` 实用程序取代。
* 所有基于 ID 的 SQL 函数列定义的数据类型均已从 `TEXT` 更改为 `NAME`。
* 从 `synchdb_add_conninfo()` 中移除了 `rulefile` 参数。
* 优化 DML 解析器，使用缓存的哈希表作为缓存，而不是在每次发生变更事件时访问目录并重建哈希。
* DML 解析器现在使用更高效的 JSONB API 调用来提升性能。
* 在 `synchdb_get_stats` 中添加了上一个批次的第一个和最后一个变更事件的源时间戳、dbz 时间戳和 pg 时间戳。这些时间戳代表了数据生成、dbz 处理和 pg 处理之间的时间。
* 优化 JNI 调用，使用缓存的 JNI 对象，而不是在每次发生变更事件时重新创建。
* 增强了 Synchdb 将批次标记为完成的方式，只需标记批次中的第一个和最后一个变更事件，而不是所有事件。
* 改进了数据处理引擎，使其在设计上更加模块化，以便能够处理更复杂的数据类型映射。
* 现在根据非原生数据类型的类别进行处理，而不是将它们全部视为文本。

### **已修复**

* 修复了 SPI 更新或删除操作因在 WHERE 子句中仅包含主键字段而无法更新或删除的问题。
* 修复了 ALTER TABLE 尝试添加重复主键时出错的问题。
* 修复了 synchdb 偶尔会从 json 更改事件中错误地查找列的架构值，从而导致后续处理失败的问题。
* 修复了“skip on error”模式下由于错误堆栈溢出而导致崩溃的问题。
* 修复了 ALTER TABLE ADD 和 DROP 列中未意识到列名可能已映射到 PostgreSQL 中的其他值的问题。

## **[SynchDB 1.0](https://github.com/Hornetlabs/synchdb/releases/tag/v1.0) - 2024-12-24**

此版本侧重于 v1.0 Beta1 版本之后的错误修复和性能增强，使其在中高数据负载下更易于使用。更多与 Debezium 调优相关的参数也已作为 PostgreSQL GUC 公开，允许用户使用不同的参数进行测试。

### **添加**

* 在 DML 解析阶段添加了数据缓存，以防止频繁访问 PostgreSQL catalog 以获取表的元组描述符结构。
* 添加 `synchdb_start_engine_bgw(name, mode)` 的变体，它接受第二个参数来指示启动连接器时使用的自定义快照模式。[此处](../user-guide/utility_functions)有更多详细信息。
* 添加多个可以调整 Debezium Runner 性能的 GUC。完整列表请参阅[此处](../user-guide/configuration.md)。
* 添加一个调试 SQL 函数 `synchdb_log_jvm_meminfo(name)`，该函数使指定的连接器在 PostgreSQL 日志文件中输出当前 JVM 堆内存使用情况摘要。
* 添加一个新视图 `synchdb_stats_view`，用于打印所有连接器的统计信息。更多详细信息请参阅[此处](../user-guide/utility_functions)。
* 增加了新的SQL函数 `synchdb_reset_stats(name)` 来清除指定连接器的统计信息。更多详细信息请参阅[此处](../user-guide/utility_functions)。
* 添加一个测试数据创建脚本，用于快速生成 MySQL 数据库类型的测试表和数据。

### **变更**

* synchdb_state_view()：添加了一个名为 `stage` 的新字段，用于标识 connector 的当前阶段（值可以是 `Initial Snapshot` 或 `Change Data Capture`）。
* synchdb_state_view()：仅显示有效连接器的状态。
* 删除了在发生错误时向 Debezium Runner 发送 “部分批处理完成” 通知，因为批处理现在由一个 PostgreSQL 事务处理，且不允许部分完成。
* 现在可以在 [规则文件](../user-guide/transform_rule_file) 中指定 connector 所需的 SSL 相关参数。
* 现在可以通过 GUC 配置分配给运行 Debezium Runner 的 JVM 的最大堆内存。
* 现在可以配置连接器后台进程的最大数量，而不是硬编码的 30。

### **修复**

* 在 Debezium runner 内添加节流控制，修复了 Debezium Runner 在数据量大时内存快速积累问题。
* 解决了 SynchDB 和 Debezium runner 组件中的大部分内存泄漏问题。
* 更正了 SynchDB 中 memory context 的使用，以便在每次更改事件处理结束时正确释放内存。
* 通过在单个 PostgreSQL 事务中处理一批改动，显著提高了 SynchDB 的处理速度。
* 将 SQLServer 的 `char` 类型的默认数据类型大小映射从 0 更正为 -1。
* 解决了使用 SPI 进行 DML 处理期间内存使用率过高的问题。

## [SynchDB 1.0 Beta1](https://github.com/Hornetlabs/synchdb/releases/tag/v1.0_beta1) - 2024-10-23

第一版 SynchDB 软件发布，为从异构数据库到 PostgreSQL 的无缝复制奠定了坚实的基础

### **新增**

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

### **变更**

### **修复**