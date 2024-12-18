---
weight: 110
---
# 函数参考
## 连接器管理

### synchdb_add_conninfo

**用途**: 创建新的连接器配置

**参数**:

| 参数 | 说明 | 必填 | 示例 | 注意事项 |
|:-:|:-|:-:|:-|:-|
| `name` | 此连接器的唯一标识符 | ✓ | `'mysqlconn'` | 必须在所有连接器中唯一 |
| `hostname` | 异构数据库的IP地址或主机名 | ✓ | `'127.0.0.1'` | 支持IPv4、IPv6和主机名 |
| `port` | 数据库连接端口号 | ✓ | `3306` | 默认值: MySQL(3306), SQLServer(1433) |
| `username` | 身份验证用户名 | ✓ | `'mysqluser'` | 需要适当的权限 |
| `password` | 身份验证密码 | ✓ | `'mysqlpwd'` | 安全存储 |
| `source database` | 源数据库名称 | ✓ | `'inventory'` | 必须存在于源系统中 |
| `destination database` | 目标PostgreSQL数据库 | ✓ | `'postgres'` | 必须存在于PostgreSQL中 |
| `table` | 表规范模式 | ☐ | `'[db].[table]'` | 空=复制所有表，支持正则表达式（例如，mydb.testtable*），使用 `file:` 前缀使连接器从 JSON 文件读取表列表（例如，file:/path/to/filelist.json）。文件格式见下文 |
| `connector` | 连接器类型(`mysql`/`sqlserver`) | ✓ | `'mysql'` | 参见上述支持的连接器 |
| `rule file` | 数据类型转换规则 | ☐ | `'myrule.json'` | 必须位于$PGDATA目录中 |

**Table列表文件示例**:
```json
{
    "table_list":
    [
        "mydb.table1",
        "mydb.table2",
        "mydb.table3",
        "mydb.table4"
    ]
}
```

**使用示例**:
```sql
-- MySQL示例
SELECT synchdb_add_conninfo(
    'mysqlconn',    -- 连接器名称
    '127.0.0.1',    -- 主机
    3306,           -- 端口
    'mysqluser',    -- 用户名
    'mysqlpwd',     -- 密码
    'inventory',    -- 源数据库
    'postgres',     -- 目标数据库
    '',             -- 表（空表示全部）
    'mysql',        -- 连接器类型
    'myrule.json'   -- 规则文件
);

-- SQL Server示例
SELECT synchdb_add_conninfo(
    'sqlserverconn',
    '127.0.0.1',
    1433,
    'sa',
    'MyPassword123',
    'testDB',
    'postgres',
    'dbo.orders',   -- 指定表
    'sqlserver',
    'mssql_rules.json'
);
```

### 基本控制函数

#### synchdb_start_engine_bgw
**用途**: 启动连接器
```sql
SELECT synchdb_start_engine_bgw('mysqlconn');
```

您还可以包含快照模式来启动连接器，否则将默认使用“initial”模式。请参阅下面的不同快照模式列表。
```sql
SELECT synchdb_start_engine_bgw('mysqlconn', 'no_data');
```

#### synchdb_pause_engine
**用途**: 暂停运行中的连接器
```sql
SELECT synchdb_pause_engine_bgw('mysqlconn');
```

#### synchdb_resume_engine
**用途**: 恢复已暂停的连接器
```sql
SELECT synchdb_resume_engine('mysqlconn');
```

#### synchdb_stop_engine_bgw
**用途**: 终止连接器
```sql
SELECT synchdb_stop_engine('mysqlconn');
```

### synchdb_log_jvm_meminfo
**用途**: 使 Java 虚拟机 (JVM) 输出当前heap和non-heap使用情况统计信息到 PostgreSQL log 文件。
```sql
SELECT synchdb_log_jvm_meminfo('mysqlconn');
```

检查 PostgreSQL 日志文件：
```
2024-12-09 14:34:21.910 PST [25491] LOG:  Requesting memdump for mysqlconn connector
2024-12-09 14:34:21 WARN  DebeziumRunner:297 - Heap Memory:
2024-12-09 14:34:21 WARN  DebeziumRunner:298 -   Used: 19272600 bytes
2024-12-09 14:34:21 WARN  DebeziumRunner:299 -   Committed: 67108864 bytes
2024-12-09 14:34:21 WARN  DebeziumRunner:300 -   Max: 2147483648 bytes
2024-12-09 14:34:21 WARN  DebeziumRunner:302 - Non-Heap Memory:
2024-12-09 14:34:21 WARN  DebeziumRunner:303 -   Used: 42198864 bytes
2024-12-09 14:34:21 WARN  DebeziumRunner:304 -   Committed: 45023232 bytes
2024-12-09 14:34:21 WARN  DebeziumRunner:305 -   Max: -1 bytes

```

## 状态管理

### synchdb_state_view
**用途**: 监控连接器状态

```sql
SELECT * FROM synchdb_state_view();
```

**返回字段**:

| 字段 | 说明 | 类型 |
|-|-|-|
| `id` | 连接器槽标识符 | Integer |
| `connector` | 连接器类型(`mysql`或`sqlserver`) | Text |
| `name` | 关联的连接器名称 | Text |
| `pid` | 工作进程ID | Integer |
| `state` | 当前连接器状态 | Text |
| `err` | 最新错误消息 | Text |
| `last_dbz_offset` | 最后记录的Debezium偏移量 | JSON |

**可能的状态**:

- 🔴 `stopped` - 已停止
- 🟡 `initializing` - 初始化中
- 🟠 `paused` - 已暂停
- 🟢 `syncing` - 主动轮询中
- 🔵 `parsing` - 解析事件中
- 🟣 `converting` - 转换数据中
- ⚪ `executing` - 应用更改中
- 🟤 `updating offset` - 更新检查点中
- 🟨 `restarting` - 重启中
- ⚪ `dumping memory` - 正在输出 JVM 内存信息到 log 文件
- ⚫ `unknown` - 未知状态

### synchdb_stats_view
**用途**：累计收集连接器处理统计信息

```sql
SELECT * FROM synchdb_stats_view();
```

| 字段 | 说明 | 类型 |
|-|-|-|
| name | 关联的连接器名称 | Text |
| ddls | 已完成的 DDL 操作数 | Bigint |
| dmls | 已完成的 DML 操作数 | Bigint |
| reads | 初始快照阶段完成的 READ 事件数 | Bigint |
| creating | CDC 阶段完成的 CREATES 事件数 | Bigint |
| updates | CDC 阶段完成的 UPDATES 事件数 | Bigint |
| deletes | CDC 阶段完成的 DELETES 事件数 | Bigint |
| bad_events | 忽略的坏事件数（例如空事件、不支持的 DDL 事件等）| Bigint |
| total_events |处理的事件总数（包括 bad_events） | Bigint |
| batches_done | 完成的批次数 | Bigint |
| avg_batch_size | 平均批次大小（total_events / batches_done） | Bigint |

### synchdb_reset_stats
**用途**：重置指定连接器的所有统计信息

```sql
SELECT synchdb_reset_stats('mysqlconn');
```

### synchdb_set_offset
**用途**: 配置自定义起始位置

**MySQL示例**:
```sql
SELECT synchdb_set_offset(
    'mysqlconn', 
    '{"ts_sec":1725644339,"file":"mysql-bin.000004","pos":138466,"row":1,"server_id":223344,"event":2}'
);
```

**SQL Server示例**:
```sql
SELECT synchdb_set_offset(
    'sqlserverconn',
    '{"event_serial_no":1,"commit_lsn":"00000100:00000c00:0003","change_lsn":"00000100:00000c00:0002"}'
);
```

## 快照管理

### synchdb_restart_connector
**用途**: 使用指定的快照模式重新初始化连接器

**快照模式**:

| 模式 | 说明 | 使用场景 |
|:-:|-|-|
| `always` | 每次启动时执行完整快照 | 完整数据验证 |
| `initial` | 仅首次快照 | 正常操作 |
| `initial_only` | 执行一次快照后停止 | 数据迁移 |
| `no_data` | 仅捕获表结构，不含数据 | 架构同步 |
| `never` | 跳过快照，直接开始流式传输 | 实时更新 |
| `recovery` | 从源表重建主题 | 灾难恢复 |
| `when_needed` | 按需执行快照 | 自动恢复 |

**示例**:
```sql
-- 使用特定快照模式重启
SELECT synchdb_restart_connector('mysqlconn', 'initial');
```

---
📝 **附加说明**:

- 启动前始终验证连接器配置
- 在快照操作期间监控系统资源
- 在进行重要操作前备份PostgreSQL目标数据库
- 测试从PostgreSQL服务器到源数据库的连接性
- 确保源数据库已配置所需权限
- 建议定期监控错误日志
