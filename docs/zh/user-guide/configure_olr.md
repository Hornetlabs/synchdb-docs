# 为 Oracle 配置 Openlog Replicator

## **概述**

除了基于 LogMiner 的 Oracle 复制功能外，**SynchDB** 还支持将 **Openlog Replicator (OLR)** 作为基于 Debezium 的 Oracle 连接器（类型为 `oracle`）的数据源。此集成支持通过独立的复制服务器从 Oracle 数据库捕获基于重做日志的低延迟变更数据。

此外，Synchdb 还支持原生 Openlog Replicator 连接器，该连接器不依赖于 Debezium。这是一个纯 C 语言编写的 OLR 客户端，可消除 JNI 开销。

本指南详细介绍了以下配置步骤：

* 将 Oracle 连接器与 OLR 端点关联
* 或创建原生 Openlog Replicator 连接器

这两个操作都可以使用 `synchdb_add_olr_conninfo()` 和 `synchdb_del_olr_conninfo()` 完成。

## **要求**

- **Openlog Replicator 版本**：`1.3.0`（已验证与 Debezium 2.7.x 的兼容性）
- 具有可供 OLR 访问的重做日志的 Oracle 实例
- Openlog Replicator 必须已配置并正在运行
- SynchDB 中现有的 Oracle 连接器（使用 `synchdb_add_conninfo()` 创建）

有关通过 Docker 部署 Openlog Replicator 的详细信息，请参阅此[外部指南](https://highgo.atlassian.net/wiki/external/OTUzY2Q2OWFkNzUzNGVkM2EyZGIyMDE1YzVhMDdkNWE)。

## **`synchdb_add_conninfo()`**

要创建**基于 Debezium 的**或**原生** Openlog Replicator，用户首先必须使用 `synchdb_add_conninfo()` 创建连接器，但要选择不同的“连接器类型”。例如：

```sql
SELECT synchdb_add_conninfo('olrconn',
							'ora19c',
							1521,
							'DBZUSER',
							'dbz',
							'FREE',
							'postgres',
							'null',
							'null',
							'olr'); -- 原生 Openlog Replicator 连接器为“olr”，基于 Debezium 的 Openlog Replicator 连接器为“oracle”。

```

更多关于创建连接器的信息，请访问[此处](https://docs.synchdb.com/user-guide/create_a_connector/)

## **`synchdb_add_olr_conninfo()`**

创建连接器后，用户可以为现有的 Oracle 连接器注册一个 Openlog Replicator 端点。 

**签名：**

```sql
synchdb_add_olr_conninfo(
	conn_name TEXT, -- 连接器名称
	olr_host TEXT, -- OLR 实例的主机名或 IP
	olr_port INT, -- OLR 公开的端口号（通常为 7070）
	olr_source TEXT -- OLR 中配置的 Oracle 源名称
)
```

**示例：**

这将指示 SynchDB 使用 Oracle 源标识符 ORACLE，从运行于 10.55.13.17:7070 的 Openlog Replicator 实例流式传输连接器 `oracleconn` 的变更。调用 `synchdb_start_engine_bgw` 启动此连接器。

```sql
SELECT synchdb_add_olr_conninfo('oracleconn', '10.55.13.17', 7070, 'ORACLE');

```

## **synchdb_del_olr_conninfo**

删除特定连接器的 OLR 配置，将其恢复为使用 LogMiner。

**签名：**

```sql
synchdb_del_olr_conninfo(conn_name TEXT)

```

**示例：**

此命令禁用 oracleconn 使用 OLR。使用 `synchdb_start_engine_bgw` 启动连接器将恢复为默认的日志挖掘策略。

```sql
SELECT synchdb_del_olr_conninfo('oracleconn');

```

## **行为说明**

* 当 LogMiner 和 OLR 配置同时存在时，SynchDB 默认使用 Openlog Replicator 进行更改捕获。
* 修改连接器的 OLR 配置后，需要重新启动连接器。
* Oracle 实例和 OLR 必须在 SCN 进程和 resetlogs 标识方面保持同步，以保持一致性。
* 如果 OLR 连接信息存在，且连接器类型为“oracle”，则它将使用基于 Debezium 的 Openlog Replicator 客户端来传输更改。
* 如果 OLR 连接信息存在，且连接器类型为“olr”，则它将使用原生 Openlog Replicator 客户端来传输更改。
* 有关 Openlog Replicator 的更多信息，请访问[此处](https://github.com/bersler/OpenLogReplicator)