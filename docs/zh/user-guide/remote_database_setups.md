---
weight: 15
---
# 异构源数据库设置

## 为 SynchDB 设置 MySQL

### 创建用户

创建用户
```sql
mysql> CREATE USER 'user'@'localhost' IDENTIFIED BY 'password';
```

授予所需权限
```sql
mysql> GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'user' IDENTIFIED BY 'password';
```

确定用户的权限
```sql
mysql> FLUSH PRIVILEGES;
```

### 启用 binlog

检查 binlog 是否启用
```sql
// for MySQL 5.x
mysql> SELECT variable_value as "BINARY LOGGING STATUS (log-bin) ::"
FROM information_schema.global_variables WHERE variable_name='log_bin';

// for MySQL 8.x
mysql> SELECT variable_value as "BINARY LOGGING STATUS (log-bin) ::"
FROM performance_schema.global_variables WHERE variable_name='log_bin';
```

如果 binlog 处于“OFF”状态，则将以下属性添加到配置文件中
```
server-id         			= 223344
log_bin                     = mysql-bin
binlog_format               = ROW
binlog_row_image            = FULL
binlog_expire_logs_seconds  = 864000
```

再次检查binlog状态
```sql
// for MySQL 5.x
mysql> SELECT variable_value as "BINARY LOGGING STATUS (log-bin) ::"
FROM information_schema.global_variables WHERE variable_name='log_bin';

// for MySQL 8.x
mysql> SELECT variable_value as "BINARY LOGGING STATUS (log-bin) ::"
FROM performance_schema.global_variables WHERE variable_name='log_bin';
```

### ### 启用 GTID（可选）
全局事务标识符 (GTID) 唯一地标识集群内服务器上发生的事务。虽然 SynchDB 连接器不强制需要，但使用 GTID 可以简化复制，并使您能够更轻松地确认主服务器和副本服务器是否一致。

启用 `gtid_mode`
```sql
mysql> gtid_mode=ON
```

启用 `enforce_gtid_consistency`
```sql
mysql> enforce_gtid_consistency=ON
```

确认更改
```
mysql> show global variables like '%GTID%';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| enforce_gtid_consistency | ON    |
| gtid_mode                | ON    |
+--------------------------+-------+
```

### 配置会话超时
当为大型数据库制处理 initial snapshot时，您建立的连接可能会在读取表时超时。您可以通过在 MySQL 配置文件中配置`interactive_timeout`和`wait_timeout`来防止此行为。

配置 `interactive_timeout:`
```sql
mysql> interactive_timeout=<duration in seconds>
```

配置 `wait_timeout`
```sql
mysql> wait_timeout=<duration in seconds>
```

### 启用查询日志事件
您可能希望查看每个二进制日志事件的原始 SQL 语句。在 MySQL 配置文件中启用`binlog_rows_query_log_events`选项允许您执行此操作。目前 SynchDB 不会以任何方式处理或解析原始 SQL 语句，即使它们包含在 binlog 内。这些仅供参考/调试

启用 `binlog_rows_query_log_events `
```sql
mysql> binlog_rows_query_log_events=ON
```

### 验证 Binlog 行值选项
验证数据库中 `binlog_row_value_options` 变量的设置。要使连接器能够使用 UPDATE 事件，必须将此变量设置为除 `PARTIAL_JSON` 以外的值。

check current variable value
```sql
mysql> show global variables where variable_name = 'binlog_row_value_options';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| binlog_row_value_options |       |
+--------------------------+-------+
```

如果变量的值为“PARTIAL_JSON”，请运行以下命令取消设置它
```sql
mysql> set @@global.binlog_row_value_options="" ;
```


## 为 SynchDB 设置 SQLServer

### 在 SQLServer 数据库上启用 CDC
在为表启用 CDC 之前，必须先为 SQL Server 数据库启用它。SQLServer 管理员通过运行系统存储过程来启用 CDC。可以使用 SQL Server Management Studio 或 Transact-SQL 来运行系统存储过程。

```sql
USE MyDB
GO
EXEC sys.sp_cdc_enable_db
GO
```

### 在 SQLServer 表上启用 CDC
SQLServer 管理员必须在您希望 SynchDB 捕获的源表上启用变更数据捕获。数据库必须已启用 CDC。要在表上启用 CDC，SQLServer 管理员需要为表运行存储过程“sys.sp_cdc_enable_table”。必须为要捕获的每个表启用 SQL Server CDC。

为 SynchDB 启用 3 个表`customer`、`district` 和 `history` 以进行捕获：

```sql
USE MyDB
GO
EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', @source_name = 'customer', @role_name = NULL, @supports_net_changes = 0;
EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', @source_name = 'district', @role_name = NULL, @supports_net_changes = 0;
EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', @source_name = 'history', @role_name = NULL, @supports_net_changes = 0;
GO
```

### 验证用户对 CDC 表的权限
SQLServer 管理员可以运行系统存储过程来查询数据库或表以检索其 CDC 配置信息。

以下查询返回数据库中启用 CDC 的每个表的配置信息，其中包含调用者有权访问的更改数据。如果结果为空，请验证用户是否有权访问捕获实例和 CDC 表。

```sql
USE MyDB;
GO
EXEC sys.sp_cdc_help_change_data_capture
GO
```

### 当启用 CDC 时表架构发生更改
如果某个表已添加到 CDC 捕获列表并已被 SynchDB 捕获，则需要将 SQLServer 上此表发生的任何架构更改重新添加回 CDC 捕获列表，以向 SynchDB 生成正确的 DDL ALTER TABLE 事件。有关更多信息，请参阅 [DDL 复制](https://docs.synchdb.com/user-guide/transform_rule_file/) 页面。