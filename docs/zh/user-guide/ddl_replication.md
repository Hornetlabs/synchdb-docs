# DDL 复制

SynchDB 支持以下 DDL 命令：

* CREATE <table>
* ALTER <table> ADD COLUMN
* ALTER <table> DROP COLUMN
* ALTER <table> ALTER COLUMN
* DROP <table>

## CREATE TABLE
SynchDB 能够在 CREATE TABLE 事件中捕获以下属性：
* 表名，以全限定名称 (FQN) 表示。
* 每个列的名称。
* 每个列的数据类型。
* 如果适用，每个列的数据长度。
* 每个列的无符号约束。
* 每个列的数据是否可以为空。
* 每个列的默认值表达式。
* 主键列的列表。

目前不支持在 CREATE TABLE 时指定的其他属性。

## DROP TABLE
SynchDB 能够捕获 DROP TABLE 事件的以下属性：
* 以 FQN 表示的待删除表名。

## ALTER TABLE ADD COLUMN
SynchDB 能够在 ALTER TABLE ADD COLUMN 事件中捕获以下属性：
* 每个新增列的名称。
* 每个新增列的数据类型。
* 如果适用，每个新增列的数据长度。
* 每个新增列的无符号约束。
* 每个新增列的数据是否可以为空。
* 每个新增列的默认值表达式。
* 如果适用，包括新增列的主键列列表。

目前不支持在 ALTER TABLE ADD COLUMN 时指定的其他属性。

## ALTER TABLE DROP COLUMN
SynchDB 能够捕获 ALTER TABLE DROP COLUMN 事件的以下属性：
* 待删除的列名。

## ALTER TABLE ALTER COLUMN
SynchDB 能够在 ALTER TABLE ALTER COLUMN 事件中捕获以下属性：
* 现有列的数据类型更改。
* 现有列的类型长度更改。
* 更改或删除的默认值表达式。
* 更改或删除的 `not null` 约束。

目前不支持在 ALTER TABLE ALTER COLUMN 时指定的其他属性。

请注意，SynchDB 仅支持对现有列的基本数据类型更改。例如，从 INT 到 BIGINT 或从 VARCHAR 到 TEXT。不支持复杂的数据类型更改，例如从 TEXT 到 INT 或从 INT 到 TIMESTAMP。这是因为 PostgreSQL 在执行复杂数据类型更改时需要用户提供类型转换函数，而 SynchDB 目前无法确定具体的类型转换函数。在未来，可能会允许用户通过规则文件提供自己的类型转换函数，但目前尚不支持。


## MySQL DDL 更改事件
由于 MySQL 在 binlog 中记录了 DDL 和 DML 操作，SynchDB 可以实时复制这些 DDL 和 DML 操作。在 MySQL 端无需进行特殊设置来启用 DDL 复制。

## SQLServer DDL 更改事件
SQLServer 原生不支持 DDL 复制的流模式。在初始快照构建阶段，当连接器首次启动时，SynchDB 会构建表的模式。此阶段后，SynchDB 会尝试检测模式更改，但这些更改需要显式添加到 SQLServer 的 CDC 表列表中。

### 触发 SQLServer 的 CREATE TABLE 事件
在 SQLServer 上创建新表并将其添加到 CDC 表列表：
```
CREATE TABLE dbo.altertest (a int, b text);
GO

EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', @source_name = 'altertest', @role_name = NULL, @supports_net_changes = 0, @capture_instance='dbo_altertest_1
GO
```

该命令将表 `dbo.altertest` 添加到 CDC 表列表，并会触发 SynchDB 接收到一个 CREATE TABLE DDL 更改事件。

### 触发 ALTER TABLE 事件
如果修改了现有表（如添加、删除或更改列），需要显式更新 SQLServer 的 CDC 表列表，以便 SynchDB 能够接收到 ALTER TABLE 事件。

例如：

在 SQLServer 中修改表：
```
ALTER TABLE altertest ADD c NVARCHAR(MAX), d INT DEFAULT 0 NOT NULL, e NVARCHAR(255) NOT NULL, f INT DEFAULT 5 NOT NULL, CONSTRAINT PK_altertest PRIMARY KEY (d, f);
GO
```

禁用旧的捕获实例：
```
EXEC sys.sp_cdc_disable_table @source_schema='dbo', @source_name='altertest', @capture_instance='dbo_altertest_1';
GO
```

启用新的捕获实例：
```
EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', @source_name = 'altertest', @role_name = NULL, @supports_net_changes = 0, @capture_instance = 'dbo_altertest_2';
GO
```

添加新记录：
```
INSERT INTO altertest VALUES(1, 's', 'c', 1, 'v', 5);
GO
```

上述示例应允许 SynchDB 接收到一个 ALTER TABLE ADD COLUMN 和一个 INSERT 事件。无需重启 SQLServer 或 SynchDB 即可捕获这些事件。同样的适用于 ALTER TABLE DROP COLUMN 和 ALTER TABLE ALTER COLUMN 事件。



