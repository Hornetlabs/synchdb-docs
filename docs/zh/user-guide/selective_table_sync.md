# 指定表同步

可以选择来自远程异构数据库的特定表，以专注于复制。这可以防止资源浪费在复制不必要的表上。

## 选择所需的表并首次启动
表的选择是在连接器创建阶段通过 `synchdb_add_conninfo()` 完成的，我们在其中指定要复制的表列表（以FQN表示，用逗号分隔）。

例如，以下命令创建一个连接器，仅从远程MySQL数据库中复制 `inventory.orders` 和 `inventory.products` 表的更改。
```sql
SELECT synchdb_add_conninfo(
    'mysqlconn',
    '192.168.1.86',
    3306,
    'mysqluser',
    'mysqlpwd',
    'inventory',
    'postgres', 
    'inventory.orders,inventory.products',
    'mysql',
    'myrule.json'
);
```

首次启动此连接器将触发初始快照的执行，并且所选的两个表的架构和数据将被复制。

```
SELECT synchdb_start_engine_bgw('mysqlconn');
```

检查连接器状态和新表：
```
postgres=# SELECT * FROM synchdb_state_view WHERE conninfo_name='mysqlconn';
 id | connector | conninfo_name |  pid   |  state  |   err    |       last_dbz_offset
----+-----------+---------------+--------+---------+----------+-----------------------------
  0 | mysql     | mysqlconn     | 807536 | syncing | no error | offset file not flushed yet
(1 row)

postgres=# SET search_path TO inventory;
SET
postgres=# \d
                 List of relations
  Schema   |        Name        |   Type   | Owner
-----------+--------------------+----------+--------
 inventory | products           | table    | ubuntu
 inventory | products_id_seq    | sequence | ubuntu
 inventory | orders             | table    | ubuntu
 inventory | orders_ididid_seq  | sequence | ubuntu
 public    | synchdb_conninfo   | table    | ubuntu
 public    | synchdb_state_view | view     | ubuntu
(6 rows)

postgres=#
```

`mysqlconn` 连接器将捕获应用于 `inventory.orders` 和 `inventory.products` 表的后续更改。

## 运行时添加更多表以进行复制
前一节中的 `mysqlconn` 已完成初始快照并获取了所选表的架构。如果我们想添加更多表进行复制，则需要通知 Debezium 引擎更新的表部分，并再次执行初始快照。操作步骤如下：

修改 `synchdb_conninfo` 表并用更多表更新表列表。在这里，我们将另一张表 `inventory.customers` 添加到表同步列表中：
```sql
UPDATE 
  synchdb_conninfo 
SET 
  data = jsonb_set(
    data, '{table}', '"inventory.orders,inventory.products,inventory.customers"'
  ) 
WHERE 
  name = 'mysqlconn';
```

使用新的快照模式（always）重启连接器：
```sql
SELECT synchdb_restart_connector('mysqlconn', 'always');
```

上述命令指示 Debezium 引擎重新执行快照，这意味着即使其中两个表已经有数据，它也会重建所有三个选定的表。

如果异构数据库类型不支持DDL复制（如SQLServer），在重新构建之前已选定的两个表的快照时，可能会出现数据冲突错误。如果是这种情况，我们可能需要在以快照模式 = 'always' 重新启动连接器之前删除或截断这些表。

现在，我们可以再次检查我们的表：
```sql
postgres=# SET search_path TO inventory;
SET
postgres=# \d
                 List of relations
  Schema   |        Name        |   Type   | Owner
-----------+--------------------+----------+--------
 inventory | products           | table    | ubuntu
 inventory | products_id_seq    | sequence | ubuntu
 inventory | orders             | table    | ubuntu
 inventory | orders_ididid_seq  | sequence | ubuntu
 inventory | customers          | table    | ubuntu
 inventory | customers_id_seq   | sequence | ubuntu
 public    | synchdb_conninfo   | table    | ubuntu
 public    | synchdb_state_view | view     | ubuntu
(8 rows)

postgres=#

```

## 快照模式

下表包含 SynchDB 支持的快照模式列表：

|  **设置** | **描述** |
|:-:|-|
| `always`       | 连接器在每次启动时执行快照。快照包括捕获表的结构和数据。快照完成后，连接器开始流式传输后续数据库更改的事件记录。|
| `initial`（默认）      | 如果尚未执行，连接器会执行数据库快照。快照完成后，连接器开始流式传输后续数据库更改的事件记录。|
| `initial_only` | 连接器执行数据库快照。快照完成后，连接器停止，并不流式传输后续数据库更改的事件记录。|
| `no_data`      | 连接器捕获所有相关表的结构，但不捕获它们包含的数据。|
| `never`        | 连接器启动时，不执行快照，而是立即开始流式传输后续数据库更改的事件记录。|
| `recovery`     | 设置此选项以恢复丢失或损坏的数据库架构历史。在重新启动后，连接器执行快照，以从源表重建主题。|
| `when_needed`  | 连接器启动后，仅在检测到以下情况时执行快照：<br><ul><li>无法检测到任何主题偏移量</li><li>先前记录的偏移量指定的日志位置在服务器上不可用</li></ul>|
