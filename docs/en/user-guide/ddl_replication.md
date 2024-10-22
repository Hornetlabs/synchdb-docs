# DDL Replication
SynchDB supports the following DDL commands:

* CREATE [table]
* ALTER [table] ADD COLUMN
* ALTER [table] DROP COLUMN
* ALTER [table] ALTER COLUMN
* DROP [table]

## CREATE TABLE
SynchDB is able to capture the following properties during a CREATE TABLE event:
* table name expressed in Fully Qualified Name (FQN).
* name of each column.
* data type of each column.
* data length of each column if applicable. 
* unsigned constraint of each column.
* whether each column data is nullable.
* default value expression of each column.
* primary key column lists.

Other properties that can be specified during CREATE TABLE are not supported at the moment.

## DROP TABLE
SynchDB is able to capture the DROP TABLE event with the property below:
* table name expressed in FQN to be dropped

## ALTER TABLE ADD COLUMN
SynchDB is able to capture the following properties during a ALTER TABLE ADD COLUMN event:
* name of each added column.
* data type of each added column.
* data length of each added column if applicable. 
* unsigned constraint of each added column.
* whether each added column data is nullable.
* default value expression of each added column.
* primary key column lists including the added column if applicable.

Other properties that can be specified during ALTER TABLE ADD COLUMN  are not supported at the moment.

## ALTER TABLE DROP COLUMN
SynchDB is able to capture the ALTER TABLE DROP COLUMN event with the property below:
* column names to be dropped.

## ALTER TABLE ALTER COLUMN
SynchDB is able to capture the following properties during a ALTER TABLE ALTER COLUMN event:
* Altered data type to an existing column.
* Altered type length to an existing column.
* Altered or dropped default value expression.
* Altered or dropped `not null` constraint.

Other properties that can be specified during ALTER TABLE ALTER COLUMN  are not supported at the moment.

Please note that SynchDB only supports basic data type change on an existing column. For example, from INT to BIGINT or VARCHAR to TEXT. Complex data type changes such as TEXT to INT or INT to TIMESTAMP are not currently supported. This is because PostgreSQL requires the user to additioanlly supply a type casting function to perform the type casting as the result of complex data type change. SynchDB currently has to knowledge what type casting functions to use for specific type conversion. In the future, We may allow user to supply his or her own casting functions to use for specific type conversions via the rule file, but for now, it is not supported.

## MySQL DDL Change Events
Since MySQL logs both DDL and DML operations in the binlog, SynchDB is able to replicate both DDLs and DMLs as they happen. No special actions are needed on MySQL side to enable DDLs replication.

## SQLServer DDL Change Events 
SQLServer does not natively supports DDL replication in streaming mode. The table schema is constructed by SynchDB during initial snapshot construction phase when the connector is started for the very first time. After this phase, SynchDB will try to detect any schema changes but they need to be explicitly added to SQL server's CDC table list.

### Trigger CREATE TABLE event on SQLServer
To create a new table on SQL Server and added to its CDC table list:
```
CREATE TABLE dbo.altertest (a int, b text);
GO

EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', @source_name = 'altertest', @role_name = NULL, @supports_net_changes = 0, @capture_instance='dbo_altertest_1
GO
```

The command adds the table `dbo.altertest` to the CDC table list and would cause SynchDB to receive a CREATE TABLE DDL change event.

### Trigger ALTER TABLE Events
If an existing table is altered (add, drop or alter column), it needs to be explicitly updated to SQLServer's CDC table list, so that SynchDB will be able to receive the ALTER TABLE events.

For example:

Alter a table in SQLServer:
```
ALTER TABLE altertest ADD c NVARCHAR(MAX), d INT DEFAULT 0 NOT NULL, e NVARCHAR(255) NOT NULL, f INT DEFAULT 5 NOT NULL, CONSTRAINT PK_altertest PRIMARY KEY (d, f);
GO
```

Disable the old capture instance:
```
EXEC sys.sp_cdc_disable_table @source_schema='dbo', @source_name='altertest', @capture_instance='dbo_altertest_1';
GO
```

Enable as a new capture intance:
```
EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', @source_name = 'altertest', @role_name = NULL, @supports_net_changes = 0, @capture_instance = 'dbo_altertest_2';
GO
```

Add new record:
```
INSERT INTO altertest VALUES(1, 's', 'c', 1, 'v', 5);
GO
```

The above example should allow SynchDB to receive an ALTER TABLE ADD COLUMN and an INSERT events. There is no need to restart both SQL Server or SynchDB to capture such events. The same apply to ALTER TABLE DROP COLUMN and ALTER TABLE ALTER COLUMN events as well.



