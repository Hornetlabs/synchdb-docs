# 为 Oracle 配置 Openlog Replicator

## **概述**

除了基于 LogMiner 的 Oracle 复制功能外，**SynchDB** 还支持使用 **Openlog Replicator (OLR)** 从 Oracle 数据库进行流式传输。SynchDB 支持两种类型的 Openlog Replicator：

1. 基于 Debezium 的 Openlog Replicator
2. 原生 Openlog Replicator（不通过 Debezium）- BETA 版

这两种 Openlog Replicator 都需要根据以下设置示例进行配置，并在 SynchDB 进行流式传输之前连接到 Oracle。

## **要求**

- **Openlog Replicator 版本**：`1.3.0`（已验证与 Debezium 2.7.x 的兼容性）
- 具有可供 OLR 访问的重做日志的 Oracle 实例
- 必须为 OLR 授予额外权限。有关具体的权限要求，请参阅[此处](https://docs.synchdb.com/zh/getting-started/remote_database_setups/)。
- Openlog Replicator 必须已配置并正在运行
- SynchDB 中现有的 Oracle 连接器（使用 `synchdb_add_conninfo()` 创建）

有关通过 Docker 部署 Openlog Replicator 的详细信息，请参阅此[外部指南](https://highgo.atlassian.net/wiki/external/OTUzY2Q2OWFkNzUzNGVkM2EyZGIyMDE1YzVhMDdkNWE)。

## **Openlog Replicator 配置示例**

SynchDB 的 OLR 支持基于以下配置示例构建。

**Version 1.3.0**
```json
{
  "version": "1.3.0",
  "source": [
    {
      "alias": "SOURCE",
      "name": "ORACLE",
      "reader": {
        "type": "online",
        "user": "DBZUSER",
        "password": "dbz",
        "server": "//ora19c:1521/FREE"
      },
      "format": {
        "type": "json",
        "column": 2,
        "db": 3,
        "interval-dts": 9,
        "interval-ytm": 4,
        "message": 2,
        "rid": 1,
        "schema": 7,
        "timestamp-all": 1,
        "scn-all": 1
      },
      "memory": {
        "min-mb": 64,
        "max-mb": 1024
      },
      "filter": {
        "table": [
          {"owner": "DBZUSER", "table": ".*"}
        ]
      },
      "flags": 32
    }
  ],
  "target": [
    {
      "alias": "SYNCHDB",
      "source": "SOURCE",
      "writer": {
        "type": "network",
        "uri": "0.0.0.0:7070"
      }
    }
  ]
}

```

**Version 1.8.5**
```json
{
  "version": "1.8.5",
  "source": [
    {
      "alias": "SOURCE",
      "name": "ORACLE",
      "reader": {
        "type": "online",
        "user": "DBZUSER",
        "password": "dbz",
        "server": "//ora19c:1521/FREE"
      },
      "format": {
        "type": "json",
        "column": 2,
        "db": 3,
        "interval-dts": 9,
        "interval-ytm": 4,
        "message": 2,
        "rid": 1,
        "schema": 7,
        "timestamp-all": 1,
        "scn-type": 1
      },
      "memory": {
        "min-mb": 64,
        "max-mb": 1024,
        "swap-path": "/opt/OpenLogReplicator/olrswap"
      },
      "filter": {
        "table": [
          {"owner": "DBZUSER", "table": ".*"}
        ]
      },
      "flags": 32
    }
  ],
  "target": [
    {
      "alias": "DEBEZIUM",
      "source": "SOURCE",
      "writer": {
        "type": "network",
        "uri": "0.0.0.0:7070"
      }
    }
  ]
}

```

请注意以下几点：

- "source"."name": "ORACLE" -> 通过 `synchdb_add_olr_conninfo()` 定义 OLR 参数时，此字段应与 `olr_source` 值匹配（见下文）
- "source"."reader"."user" -> 通过 `synchdb_add_conninfo()` 创建连接器时，此字段应与 `username` 值匹配
- "source"."reader"."password" -> 通过 `synchdb_add_conninfo()` 创建连接器时，此字段应与 `password` 值匹配
- "source"."reader"."server" -> 通过 `synchdb_add_conninfo()` 创建连接器时，此字段应包含 `hostname`、`port` 和 `source database` 的值
- "source"."filter"."table":[] -> 这会过滤 Openlog Replicator 捕获的变更事件。<<<**重要**>>>：这是目前从 Oracle 过滤变更事件的唯一方法，因为 SynchDB 中的 OLR 实现目前不进行任何过滤。（通过 `synchdb_add_conninfo()` 创建连接器时，`table` 和 `snapshot table` 的值会被忽略。）
- "format":{} -> 基于 Debezium 或原生 Openlog Replicator 连接器提取的特定 Paylod 格式。请按照指定的方式使用这些值。
- “memory”.“swap-path” -> 这告诉 OLR 在内存不足的情况下将交换文件写入哪里。
- "target".[0].."writer"."type": -> 必须指定 `network`，因为 Debezium 和原生 Openlog Replicator 连接器都通过网络与 Openlog Replicator 通信。
- "target".[0].."writer"."uri": -> 这是 Openlog Replicator 监听的绑定主机和端口，SynchDB 应该能够在通过 `synchdb_add_olr_conninfo()` 定义 OLR 参数时通过 `olr_host` 和 `olr_port` 访问该主机和端口。

## **`synchdb_add_conninfo()`**

要创建**原生** OLR 连接器（使用 `type` = 'olr'）：

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
                            'oracle');

```

要创建**基于 Debezium 的** OLR 连接器（使用 `type` = 'oracle'）：

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
                            'olr');

```

<<<**重要**>>> **SynchDB 必须使用标志 (WITH_OLR=1) 进行编译和构建，以支持本机 openlog 复制器连接器。**

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

这将指示 SynchDB 使用 Oracle 源标识符 ORACLE，从运行于 olrhost:7070 的 Openlog Replicator 实例流式传输连接器 `olrconn` 的变更。调用 `synchdb_start_engine_bgw` 启动此连接器。

```sql
SELECT synchdb_add_olr_conninfo('olrconn', 'olrhost', 7070, 'ORACLE');

```

## **synchdb_del_olr_conninfo**

删除特定连接器的 OLR 配置，将其恢复为使用 LogMiner。

**签名：**

```sql
synchdb_del_olr_conninfo(conn_name TEXT)

```

**示例：**

此命令禁用 oracleconn 的 OLR 使用。使用 `synchdb_start_engine_bgw` 启动连接器将回退到默认的日志挖掘策略（基于 Debezium 的 OLR 连接器）。如果使用原生 Openlog Replicator 连接器，缺少 OLR 配置将导致连接器启动时出错。

```sql
SELECT synchdb_del_olr_conninfo('olrconn');

```

## **基于 Debezium 的 Openlog Replicator 连接器的行为说明**

* 当 LogMiner 和 OLR 配置都存在时，SynchDB 默认使用 Openlog Replicator 进行变更捕获。
* 如果 OLR 配置缺失，SynchDB 将使用日志挖掘策略来流式传输变更。
* 修改 OLR 配置后，需要重新启动连接器。

## **基于 Debezium 的 Openlog Replicator 连接器的行为说明**

* 目前为 BETA 版本。
* SynchDB 管理与 Openlog Replicator 的连接，并在不使用 Debezium 的情况下流式传输变更。
* 需要 OLR 配置，否则连接器启动时会出错。
* 依赖 Debezium 的 Oracle 连接器完成初始快照，并在完成后关闭，后续的 CDC 由 SynchDB 内部针对 Openlog Replicator 原生完成。
* 依赖 IvorySQL 的 Oracle 解析器来处理 DDL 事件。在使用原生 openlog replicator 连接器之前，必须先编译并安装它。
* 访问[此处](https://github.com/bersler/OpenLogReplicator) 了解有关 Openlog Replicator 的更多信息。