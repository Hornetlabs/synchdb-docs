# 表快照和重新快照

## 初始快照
SynchDB 中的“初始快照”（或表快照）是指复制所有指定表的表结构和初始数据。这类似于 PostgreSQL 逻辑复制中的“表同步”。当使用默认“initial”模式启动连接器时，它将在进入变更数据捕获 (CDC) 阶段之前自动执行初始快照。可以使用“never”模式完全省略此操作，或使用“no_data”模式部分省略此操作。有关所有快照选项，请参阅[此处](https://docs.synchdb.com/zh/user-guide/start_stop_connector/)。

初始快照完成后，连接器在后续重启时将不再执行此操作，而是从上次未完成的偏移量开始继续执行 CDC。此行为由 Debezium 引擎管理的元数据文件控制。有关元数据文件的更多信息，请参阅[此处](https://docs.synchdb.com/zh/architecture/metadata_files/)。

## 重新快照
如果出于任何原因，用户需要再次执行初始快照以重建“所有指定的表”和所有初始数据，我们需要使用“始终”快照模式，这会导致连接器在启动时再次获取架构和初始数据。您可能需要删除所有指定的表或清除其中的所有数据，然后 SynchDB 才会尝试重新创建表并填充初始数据（这些数据可能已经存在）。

请谨慎操作，因为在您的设置中执行此操作可能过于激进。更好的替代方案是使用选择性快照，在“始终”快照模式下，仅对选定的表进行重新快照。请参见下文。

## 选择性快照
可以在运行时创建或更改连接器时配置选择性快照。这可以通过在“快照表”参数中指定要执行快照的表列表来实现。例如：

**创建期间：**
此示例创建一个连接器，该连接器对 `inventory.orders`、`inventory.customers` 和 `invnetory.produts` 表执行 CDC，但如果连接器以 `always` 快照模式启动，则只会对 `inventory.products` 再次执行初始快照。

```sql
SELECT synchdb_add_conninfo(
    'mysqlconn',
    '127.0.0.1',
    3306,
    'mysqluser', 
    'mysqlpwd',
    'inventory',
    'postgres', 
    'inventory.orders,inventory.customers,invnetory.produts',
    'inventory.products',
    'mysql');

SELECT synchdb_start_engine_bgw('mysqlconn', 'always');
```

**修改现有连接器：**
此示例将 `inventory.products` 设置为快照表字段。以 `always` 模式启动时，只有 `inventory.products` 表会重新创建快照。

```sql
UPDATE synchdb_conninfo
	SET data = jsonb_set(data, '{snapshottable}', 'inventory.products', true) 
	WHERE name = 'mysqlconn';

SELECT synchdb_start_engine_bgw('mysqlconn', 'always');

```