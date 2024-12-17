---
weight: 50
---
# 默认数据类型映射

默认数据类型映射是在SynchDB内部构建的哈希表，包含：
* 键：{数据类型，自动增量}
* 值：{数据类型，大小}

其中：
* 数据类型：数据类型的字符串表示，如INT、TEXT、NUMERIC
* 自动增量：表示数据类型是否具有自动增量属性的标志
* 大小：转换后的大小值。-1：无变化，0：未指定大小

默认数据类型映射条目可以通过在规则文件中定义数据类型转换规则来覆盖。更多信息请参见[转换规则文件](https://docs.synchdb.com/user-guide/transform_rule_file/)。

## MySQL 默认数据类型映射
```c
DatatypeHashEntry mysql_defaultTypeMappings[] =
{
	{{"INT", true}, "SERIAL", 0},
	{{"BIGINT", true}, "BIGSERIAL", 0},
	{{"SMALLINT", true}, "SMALLSERIAL", 0},
	{{"MEDIUMINT", true}, "SERIAL", 0},
	{{"ENUM", false}, "TEXT", 0},
	{{"SET", false}, "TEXT", 0},
	{{"BIGINT", false}, "BIGINT", 0},
	{{"BIGINT UNSIGNED", false}, "NUMERIC", -1},
	{{"NUMERIC UNSIGNED", false}, "NUMERIC", -1},
	{{"DEC", false}, "DECIMAL", -1},
	{{"DEC UNSIGNED", false}, "DECIMAL", -1},
	{{"DECIMAL UNSIGNED", false}, "DECIMAL", -1},
	{{"FIXED", false}, "DECIMAL", -1},
	{{"FIXED UNSIGNED", false}, "DECIMAL", -1},
	{{"BIT(1)", false}, "BOOLEAN", 0},
	{{"BIT", false}, "BIT", -1},
	{{"BOOL", false}, "BOOLEAN", -1},
	{{"DOUBLE", false}, "DOUBLE PRECISION", 0},
	{{"DOUBLE PRECISION", false}, "DOUBLE PRECISION", 0},
	{{"DOUBLE PRECISION UNSIGNED", false}, "DOUBLE PRECISION", 0},
	{{"DOUBLE UNSIGNED", false}, "DOUBLE PRECISION", 0},
	{{"REAL", false}, "REAL", 0},
	{{"REAL UNSIGNED", false}, "REAL", 0},
	{{"FLOAT", false}, "REAL", 0},
	{{"FLOAT UNSIGNED", false}, "REAL", 0},
	{{"INT", false}, "INT", 0},
	{{"INT UNSIGNED", false}, "BIGINT", 0},
	{{"INTEGER", false}, "INT", 0},
	{{"INTEGER UNSIGNED", false}, "BIGINT", 0},
	{{"MEDIUMINT", false}, "INT", 0},
	{{"MEDIUMINT UNSIGNED", false}, "INT", 0},
	{{"YEAR", false}, "INT", 0},
	{{"SMALLINT", false}, "SMALLINT", 0},
	{{"SMALLINT UNSIGNED", false}, "INT", 0},
	{{"TINYINT", false}, "SMALLINT", 0},
	{{"TINYINT UNSIGNED", false}, "SMALLINT", 0},
	{{"DATETIME", false}, "TIMESTAMP", -1},
	{{"TIMESTAMP", false}, "TIMESTAMPTZ", -1},
	{{"BINARY", false}, "BYTEA", 0},
	{{"VARBINARY", false}, "BYTEA", 0},
	{{"BLOB", false}, "BYTEA", 0},
	{{"MEDIUMBLOB", false}, "BYTEA", 0},
	{{"LONGBLOB", false}, "BYTEA", 0},
	{{"TINYBLOB", false}, "BYTEA", 0},
	{{"LONG VARCHAR", false}, "TEXT", -1},
	{{"LONGTEXT", false}, "TEXT", -1},
	{{"MEDIUMTEXT", false}, "TEXT", -1},
	{{"TINYTEXT", false}, "TEXT", -1},
	{{"JSON", false}, "JSONB", -1},

	/* spatial types - map to TEXT by default */
	{{"GEOMETRY", false}, "TEXT", -1},
	{{"GEOMETRYCOLLECTION", false}, "TEXT", -1},
	{{"GEOMCOLLECTION", false}, "TEXT", -1},
	{{"LINESTRING", false}, "TEXT", -1},
	{{"MULTILINESTRING", false}, "TEXT", -1},
	{{"MULTIPOINT", false}, "TEXT", -1},
	{{"MULTIPOLYGON", false}, "TEXT", -1},
	{{"POINT", false}, "TEXT", -1},
	{{"POLYGON", false}, "TEXT", -1}
};

```

## SQL Server 默认数据类型映射
```c
DatatypeHashEntry sqlserver_defaultTypeMappings[] =
{
	{{"int identity", true}, "SERIAL", 0},
	{{"bigint identity", true}, "BIGSERIAL", 0},
	{{"smallint identity", true}, "SMALLSERIAL", 0},
	{{"enum", false}, "TEXT", 0},
	{{"int", false}, "INT", 0},
	{{"bigint", false}, "BIGINT", 0},
	{{"smallint", false}, "SMALLINT", 0},
	{{"tinyint", false}, "SMALLINT", 0},
	{{"numeric", false}, "NUMERIC", 0},
	{{"decimal", false}, "NUMERIC", 0},
	{{"bit(1)", false}, "BOOL", 0},
	{{"bit", false}, "BIT", 0},
	{{"money", false}, "MONEY", 0},
	{{"smallmoney", false}, "MONEY", 0},
	{{"real", false}, "REAL", 0},
	{{"float", false}, "REAL", 0},
	{{"date", false}, "DATE", 0},
	{{"time", false}, "TIME", 0},
	{{"datetime", false}, "TIMESTAMP", 0},
	{{"datetime2", false}, "TIMESTAMP", 0},
	{{"datetimeoffset", false}, "TIMESTAMPTZ", 0},
	{{"smalldatetime", false}, "TIMESTAMP", 0},
	{{"char", false}, "CHAR", -1},
	{{"varchar", false}, "VARCHAR", -1},
	{{"text", false}, "TEXT", 0},
	{{"nchar", false}, "CHAR", 0},
	{{"nvarchar", false}, "VARCHAR", -1},
	{{"ntext", false}, "TEXT", 0},
	{{"binary", false}, "BYTEA", 0},
	{{"varbinary", false}, "BYTEA", 0},
	{{"image", false}, "BYTEA", 0},
	{{"uniqueidentifier", false}, "UUID", 0},
	{{"xml", false}, "TEXT", 0},
	/* spatial types - map to TEXT by default */
	{{"geometry", false}, "TEXT", 0},
	{{"geography", false}, "TEXT", 0},
};
```
