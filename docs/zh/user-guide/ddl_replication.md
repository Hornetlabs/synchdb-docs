# SynchDB DDL复制

## 概述
SynchDB为数据定义语言(DDL)操作提供全面支持，允许在不同数据库系统之间实现实时架构同步。

## 支持的DDL命令
SynchDB支持以下DDL操作：

✅ CREATE [table]  
✅ ALTER [table] ADD COLUMN  
✅ ALTER [table] DROP COLUMN  
✅ ALTER [table] ALTER COLUMN  
✅ DROP [table]  

## 详细命令支持
### CREATE TABLE
SynchDB在CREATE TABLE事件期间捕获以下属性：

| 属性 | 描述 |
|----------|-------------|
| **Table Name** | 完全限定名称(FQN)格式 |
| **Column Names** | 单独的列标识符 |
| **Data Types** | 列数据类型规范 |
| **Data Length** | 长度/精度规范（如适用） |
| **Unsigned Flag** | 数值类型的unsigned约束 |
| **Nullability** | NULL/NOT NULL约束 |
| **Default Values** | 默认值表达式 |
| **Primary Keys** | 主键列定义 |

> **注意**: 当前不支持其他CREATE TABLE属性

### DROP TABLE
捕获的属性：
- 要删除的表名（FQN格式）

### ALTER TABLE ADD COLUMN
捕获以下属性：

| 属性 | 描述 |
|----------|-------------|
| Column Names | 新添加列的名称 |
| Data Types | 新列的数据类型 |
| Data Length | 长度规范（如适用） |
| Unsigned Flag | Unsigned约束 |
| Nullability | NULL/NOT NULL规范 |
| Default Values | 默认值表达式 |
| Primary Keys | 更新的主键定义 |

目前不支持在ALTER TABLE ADD COLUMN期间可以指定的其他属性。

### ALTER TABLE DROP COLUMN
捕获：
- 要删除的列名列表

### ALTER TABLE ALTER COLUMN
支持的修改：

| 修改 | 描述 |
|--------------|-------------|
| Data Type | 更改列数据类型 |
| Type Length | 修改类型长度/精度 |
| Default Value | 更改/删除默认值 |
| NOT NULL | 修改/删除NOT NULL约束 |

目前不支持在ALTER TABLE ALTER COLUMN期间可以指定的其他属性。

请注意，SynchDB仅支持对现有列进行基本数据类型更改。例如，`INT` → `BIGINT`或`VARCHAR` → `TEXT`。目前不支持复杂的数据类型更改，如`TEXT` → `INT`或`INT` → `TIMESTAMP`。这是因为PostgreSQL需要用户额外提供类型转换函数来执行复杂数据类型更改的类型转换。SynchDB目前不知道对特定类型转换使用什么类型转换函数。将来，我们可能允许用户通过规则文件提供自己的转换函数用于特定类型转换，但目前尚不支持。

## 数据库特定行为
### MySQL DDL变更事件
由于MySQL在binlog中同时记录DDL和DML操作，SynchDB能够在发生时复制DDL和DML。MySQL端不需要特殊操作来启用DDL复制。

### SQLServer DDL变更事件
SQLServer在流模式下不原生支持DDL复制。表架构是在连接器首次启动时由SynchDB在初始快照构建阶段构建的。在此阶段之后，SynchDB将尝试检测任何架构更改，但需要将其显式添加到SQL Server的CDC表列表中。

#### 在SQLServer上触发CREATE TABLE事件
要在SQL Server上创建新表并将其添加到CDC表列表中：
```
CREATE TABLE dbo.altertest (a int, b text);
GO

EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', @source_name = 'altertest', @role_name = NULL, @supports_net_changes = 0, @capture_instance='dbo_altertest_1
GO
```

该命令将表`dbo.altertest`添加到CDC表列表中，并会导致SynchDB接收CREATE TABLE DDL变更事件。

#### 触发ALTER TABLE事件
如果修改现有表（添加、删除或更改列），需要将其显式更新到SQLServer的CDC表列表中，以便SynchDB能够接收ALTER TABLE事件。

例如：

在SQLServer中修改表：
```
ALTER TABLE altertest ADD c NVARCHAR(MAX), d INT DEFAULT 0 NOT NULL, e NVARCHAR(255) NOT NULL, f INT DEFAULT 5 NOT NULL, CONSTRAINT PK_altertest PRIMARY KEY (d, f);
GO
```

禁用旧的捕获实例：
```
EXEC sys.sp_cdc_disable_table @source_schema='dbo', @source_name='altertest', @capture_instance='dbo_altertest_1';
GO
```

启用为新的捕获实例：
```
EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', @source_name = 'altertest', @role_name = NULL, @supports_net_changes = 0, @capture_instance = 'dbo_altertest_2';
GO
```

添加新记录：
```
INSERT INTO altertest VALUES(1, 's', 'c', 1, 'v', 5);
GO
```

上述示例应允许SynchDB接收ALTER TABLE ADD COLUMN和INSERT事件。无需重启SQL Server或SynchDB即可捕获此类事件。这同样适用于ALTER TABLE DROP COLUMN和ALTER TABLE ALTER COLUMN事件。