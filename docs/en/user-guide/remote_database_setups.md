---
weight: 15
---
# Remote Database Setups

## Set up MySQL for SynchDB

### Create a User

create a user
```sql
mysql> CREATE USER 'user'@'localhost' IDENTIFIED BY 'password';
```

grant required permissions
```sql
mysql> GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'user' IDENTIFIED BY 'password';
```

finalize the user's permission
```sql
mysql> FLUSH PRIVILEGES;
```

### Enable binlog

check if binlog is enabled
```sql
// for MySQL 5.x
mysql> SELECT variable_value as "BINARY LOGGING STATUS (log-bin) ::"
FROM information_schema.global_variables WHERE variable_name='log_bin';

// for MySQL 8.x
mysql> SELECT variable_value as "BINARY LOGGING STATUS (log-bin) ::"
FROM performance_schema.global_variables WHERE variable_name='log_bin';
```

add the properties below to configuration file if binlog is `OFF`
```
server-id         			= 223344
log_bin                     = mysql-bin
binlog_format               = ROW
binlog_row_image            = FULL
binlog_expire_logs_seconds  = 864000
```

check binlog status again
```sql
// for MySQL 5.x
mysql> SELECT variable_value as "BINARY LOGGING STATUS (log-bin) ::"
FROM information_schema.global_variables WHERE variable_name='log_bin';

// for MySQL 8.x
mysql> SELECT variable_value as "BINARY LOGGING STATUS (log-bin) ::"
FROM performance_schema.global_variables WHERE variable_name='log_bin';
```

### Enable GTIDs (Optional)
Global transaction identifiers (GTIDs) uniquely identify transactions that occur on a server within a cluster. Though not required for SynchDB connector, using GTIDs simplifies replication and enables you to more easily confirm if primary and replica servers are consistent.

enable `gtid_mode`
```sql
mysql> gtid_mode=ON
```

enable `enforce_gtid_consistency`
```sql
mysql> enforce_gtid_consistency=ON
```

confirm changes
```
mysql> show global variables like '%GTID%';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| enforce_gtid_consistency | ON    |
| gtid_mode                | ON    |
+--------------------------+-------+
```

### Configure Session Timeout
When an initial consistent snapshot is made for large databases, your established connection could timeout while the tables are being read. You can prevent this behavior by configuring `interactive_timeout` and `wait_timeout` in your MySQL configuration file.

configure `interactive_timeout:`
```sql
mysql> interactive_timeout=<duration in seconds>
```

configure `wait_timeout`
```sql
mysql> wait_timeout=<duration in seconds>
```

### Enable Query Log Events
You might want to see the original SQL statement for each binlog event. Enabling the `binlog_rows_query_log_events` option in the MySQL configuration file allows you to do this. Currently SynchDB does not process or parse the original SQL statement in anyway even if they are included. These are for reference / debug only.

enable `binlog_rows_query_log_events `
```sql
mysql> binlog_rows_query_log_events=ON
```

### Validate Binlog Row Value Options
Verify the setting of the `binlog_row_value_options` variable in the database. To enable the connector to consume UPDATE events, this variable must be set to a value other than `PARTIAL_JSON`.

check current variable value
```sql
mysql> show global variables where variable_name = 'binlog_row_value_options';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| binlog_row_value_options |       |
+--------------------------+-------+
```

if the value of the variable is `PARTIAL_JSON`, run the following to unset it
```sql
mysql> set @@global.binlog_row_value_options="" ;
```


## Set up SQLServer for SynchDB

### Enable CDC on the SQLServer Database
Before you can enable CDC for a table, you must enable it for the SQL Server database. A SQLServer admin enables CDC by running a system stored procedure. System stored procedures can be run by using SQL Server Management Studio, or by using Transact-SQL.

```sql
USE MyDB
GO
EXEC sys.sp_cdc_enable_db
GO
```

### Enable CDC on a SQLServer Table
SQLServer admin must enable change data capture on the source tables that you want SynchDB to capture. The database must already be enabled for CDC. To enable CDC on a table, a SQLServer administrator runs the stored procedure `sys.sp_cdc_enable_table` for the table. SQL Server CDC must be enabled for every table that you want to capture.

enable 3 tables `customer`, `district`, and `history` for SynchDB to capture:

```sql
USE MyDB
GO
EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', @source_name = 'customer', @role_name = NULL, @supports_net_changes = 0;
EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', @source_name = 'district', @role_name = NULL, @supports_net_changes = 0;
EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', @source_name = 'history', @role_name = NULL, @supports_net_changes = 0;
GO
```

### Verify User Permission for CDC Tables
A SQLServer administrator can run a system stored procedure to query a database or table to retrieve its CDC configuration information. 

The query below returns configuration information for each table in the database that is enabled for CDC and that contains change data that the caller is authorized to access. If the result is empty, verify that the user has privileges to access both the capture instance and the CDC tables.

```sql
USE MyDB;
GO
EXEC sys.sp_cdc_help_change_data_capture
GO
```

### When Table Schema Changed While CDC is Enabled
If a table has already been added to the CDC capture list and being captured by SynchDB already, any schema change that has happened to this table on SQLServer needs to be re-added back to the CDC capture list to generate a proper DDL ALTER TABLE event to SynchDB. Refer to [DDL Replication](https://docs.synchdb.com/user-guide/transform_rule_file/) page for more information.