# 为 Oracle 配置 Openlog Replicator

## **概述**

除了基于 LogMiner 的复制功能外，**SynchDB** 还支持将 **Openlog Replicator (OLR)** 作为 Oracle 连接器的数据源。此集成功能支持通过独立的复制服务器从 Oracle 数据库捕获低延迟、基于重做日志的变更数据。

本指南详细介绍了使用 `synchdb_add_olr_conninfo()` 和 `synchdb_del_olr_conninfo()` 将 Oracle 连接器与 OLR 端点关联的配置步骤。

## **要求**

- **Openlog Replicator 版本**：`1.3.0`（已验证与 Debezium 2.7.x 的兼容性）
- 具有可供 OLR 访问的重做日志的 Oracle 实例
- Openlog Replicator 必须已配置并正在运行
- SynchDB 中现有的 Oracle 连接器（使用 `synchdb_add_conninfo()` 创建）

有关通过 Docker 部署 Openlog Replicator 的详细信息，请参阅此[外部指南](https://highgo.atlassian.net/wiki/external/OTUzY2Q2OWFkNzUzNGVkM2EyZGIyMDE1YzVhMDdkNWE)。

## **`synchdb_add_olr_conninfo()`**

为现有的 Oracle 连接器注册 Openlog Replicator 端点。

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