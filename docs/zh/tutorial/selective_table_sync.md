---
weight: 90
---
# 选择性表同步

可以从远程异构数据库中仅选择特定表进行同步，从而避免将资源浪费在复制不需要的表上。

## **选择所需表并首次启动**

表的选择是在连接器创建阶段通过 `synchdb_add_conninfo()` 完成的，我们指定一个要从中复制的表列表（以 FQN 表示，以逗号分隔）。

例如，以下命令创建一个连接器，该连接器仅从远程 MySQL 数据库的 `inventory.orders` 和 `inventory.products` 表复制更改。
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
    'null'
    'mysql'
);
```

首次启动此连接器将触发执行初始快照，并将复制选定的 2 个表的模式和数据。

```sql
SELECT synchdb_start_engine_bgw('mysqlconn');
```

### **验证连接器状态和表**

检查连接器状态和新表：
```sql
postgres=# Select name, state, err from synchdb_state_view;
     name      |  state  |   err
---------------+---------+----------
 mysqlconn     | polling | no error
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

快照完成后，`mysqlconn` 连接器将继续捕获 `inventory.orders` 和 `inventory.products` 表的后续更改。

## **在运行时添加更多要复制的表**

上一节中的 `mysqlconn` 已经完成了初始快照，并获取了所选表的表结构。如果我们想要添加更多要复制的表，则需要将更新的表部分通知给 Debezium 引擎，并再次执行初始快照。操作方法如下：

1. 更新 `synchdb_conninfo` 表以包含其他表。
2. 在此示例中，我们将 `inventory.customers` 表添加到同步列表：
```sql
UPDATE synchdb_conninfo
SET data = jsonb_set(data, '{table}', '"inventory.orders,inventory.products,inventory.customers"')
WHERE name = 'mysqlconn';
```
3. 配置快照表参数，使其仅包含新表 `inventory.customers`，这样 SynchDB 就不会尝试重建已完成快照的两张表。
```sql
UPDATE synchdb_conninfo
SET data = jsonb_set(data, '{snapshottable}', '"inventory.customers"')
WHERE name = 'mysqlconn';
```
4. 重启连接器，并将快照模式设置为 `always`，以执行另一次初始快照：
```sql
SELECT synchdb_restart_connector('mysqlconn', 'always');
```
这将强制 Debezium 仅对新表 `inventory.customers` 重新创建快照，同时保留旧表 `inventory.orders` 和 `inventory.products` 不变。快照完成后，所有表的 CDC 将恢复。


### **验证更新后的表**

现在，我们可以再次检查这些表：
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

## **快照模式**

SynchDB 提供不同的快照模式，具体取决于您的复制需求：

| **设置** |**描述**|
|:-:|-|
| always | 连接器每次启动时都会执行快照。快照包含已捕获表的结构和数据。快照完成后，连接器开始流式传输后续数据库更改的事件记录。|
| initial（默认）| 如果尚未执行，连接器将执行数据库快照。快照完成后，连接器开始流式传输后续数据库更改的事件记录。|
| initial_only | 连接器执行数据库快照。快照完成后，连接器将停止，并且不会流式传输后续数据库更改的事件记录。|
| no_data | 连接器捕获所有相关表的结构，但不捕获它们包含的数据。|
| never | 连接器启动时，它会立即开始流式传输后续数据库更改的事件记录，而不是执行快照。|
| recovery | 设置此选项可恢复丢失或损坏的数据库架构历史记录。重启后，连接器将运行快照，从源表重建主题|
| when_needed | 连接器启动后，仅当检测到以下情况之一时才执行快照：<br><ul><li>无法检测到任何主题偏移量</li><li>先前记录的偏移量指定的日志位置在服务器上不可用</li></ul> |
