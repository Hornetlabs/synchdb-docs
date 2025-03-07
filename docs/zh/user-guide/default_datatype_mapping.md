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

可以通过定义对象映射规则来覆盖默认的数据类型映射关系。更多信息请参见[转换规则文件](https://docs.synchdb.com/user-guide/transform_rule_file/)。

## MySQL 默认数据类型映射
```c
datatypehashentry mysql_defaulttypemappings[] =
{
	{{"int", true}, "serial", 0},
	{{"bigint", true}, "bigserial", 0},
	{{"smallint", true}, "smallserial", 0},
	{{"mediumint", true}, "serial", 0},
	{{"enum", false}, "text", 0},
	{{"set", false}, "text", 0},
	{{"bigint", false}, "bigint", 0},
	{{"bigint unsigned", false}, "numeric", -1},
	{{"numeric unsigned", false}, "numeric", -1},
	{{"dec", false}, "decimal", -1},
	{{"dec unsigned", false}, "decimal", -1},
	{{"decimal unsigned", false}, "decimal", -1},
	{{"fixed", false}, "decimal", -1},
	{{"fixed unsigned", false}, "decimal", -1},
	{{"bit(1)", false}, "boolean", 0},
	{{"bit", false}, "bit", -1},
	{{"bool", false}, "boolean", -1},
	{{"double", false}, "double precision", 0},
	{{"double precision", false}, "double precision", 0},
	{{"double precision unsigned", false}, "double precision", 0},
	{{"double unsigned", false}, "double precision", 0},
	{{"real", false}, "real", 0},
	{{"real unsigned", false}, "real", 0},
	{{"float", false}, "real", 0},
	{{"float unsigned", false}, "real", 0},
	{{"int", false}, "int", 0},
	{{"int unsigned", false}, "bigint", 0},
	{{"integer", false}, "int", 0},
	{{"integer unsigned", false}, "bigint", 0},
	{{"mediumint", false}, "int", 0},
	{{"mediumint unsigned", false}, "int", 0},
	{{"year", false}, "int", 0},
	{{"smallint", false}, "smallint", 0},
	{{"smallint unsigned", false}, "int", 0},
	{{"tinyint", false}, "smallint", 0},
	{{"tinyint unsigned", false}, "smallint", 0},
	{{"datetime", false}, "timestamp", -1},
	{{"timestamp", false}, "timestamptz", -1},
	{{"binary", false}, "bytea", 0},
	{{"varbinary", false}, "bytea", 0},
	{{"blob", false}, "bytea", 0},
	{{"mediumblob", false}, "bytea", 0},
	{{"longblob", false}, "bytea", 0},
	{{"tinyblob", false}, "bytea", 0},
	{{"long varchar", false}, "text", -1},
	{{"longtext", false}, "text", -1},
	{{"mediumtext", false}, "text", -1},
	{{"tinytext", false}, "text", -1},
	{{"json", false}, "jsonb", -1},

	/* spatial types - map to text by default */
	{{"geometry", false}, "text", -1},
	{{"geometrycollection", false}, "text", -1},
	{{"geomcollection", false}, "text", -1},
	{{"linestring", false}, "text", -1},
	{{"multilinestring", false}, "text", -1},
	{{"multipoint", false}, "text", -1},
	{{"multipolygon", false}, "text", -1},
	{{"point", false}, "text", -1},
	{{"polygon", false}, "text", -1}
};

```

## SQL Server 默认数据类型映射
```c
datatypehashentry sqlserver_defaulttypemappings[] =
{
	{{"int identity", true}, "serial", 0},
	{{"bigint identity", true}, "bigserial", 0},
	{{"smallint identity", true}, "smallserial", 0},
	{{"enum", false}, "text", 0},
	{{"int", false}, "int", 0},
	{{"bigint", false}, "bigint", 0},
	{{"smallint", false}, "smallint", 0},
	{{"tinyint", false}, "smallint", 0},
	{{"numeric", false}, "numeric", 0},
	{{"decimal", false}, "numeric", 0},
	{{"bit(1)", false}, "bool", 0},
	{{"bit", false}, "bit", 0},
	{{"money", false}, "money", 0},
	{{"smallmoney", false}, "money", 0},
	{{"real", false}, "real", 0},
	{{"float", false}, "real", 0},
	{{"date", false}, "date", 0},
	{{"time", false}, "time", 0},
	{{"datetime", false}, "timestamp", 0},
	{{"datetime2", false}, "timestamp", 0},
	{{"datetimeoffset", false}, "timestamptz", 0},
	{{"smalldatetime", false}, "timestamp", 0},
	{{"char", false}, "char", -1},
	{{"varchar", false}, "varchar", -1},
	{{"text", false}, "text", 0},
	{{"nchar", false}, "char", 0},
	{{"nvarchar", false}, "varchar", -1},
	{{"ntext", false}, "text", 0},
	{{"binary", false}, "bytea", 0},
	{{"varbinary", false}, "bytea", 0},
	{{"image", false}, "bytea", 0},
	{{"uniqueidentifier", false}, "uuid", 0},
	{{"xml", false}, "text", 0},
	/* spatial types - map to text by default */
	{{"geometry", false}, "text", 0},
	{{"geography", false}, "text", 0},
};
```

## Oracle 默认数据类型映射

```c
DatatypeHashEntry oracle_defaultTypeMappings[] =
{
	{{"binary_double", false}, "double precision", 0},
	{{"binary_float", false}, "real", 0},
	{{"float", false}, "real", 0},
	{{"number(0,0)", false}, "numeric", -1},
	{{"number(1,0)", false}, "smallint", 0},
	{{"number(2,0)", false}, "smallint", 0},
	{{"number(3,0)", false}, "smallint", 0},
	{{"number(4,0)", false}, "smallint", 0},
	{{"number(5,0)", false}, "int", 0},
	{{"number(6,0)", false}, "int", 0},
	{{"number(7,0)", false}, "int", 0},
	{{"number(8,0)", false}, "int", 0},
	{{"number(9,0)", false}, "int", 0},
	{{"number(10,0)", false}, "bigint", 0},
	{{"number(11,0)", false}, "bigint", 0},
	{{"number(12,0)", false}, "bigint", 0},
	{{"number(13,0)", false}, "bigint", 0},
	{{"number(14,0)", false}, "bigint", 0},
	{{"number(15,0)", false}, "bigint", 0},
	{{"number(16,0)", false}, "bigint", 0},
	{{"number(17,0)", false}, "bigint", 0},
	{{"number(18,0)", false}, "bigint", 0},
	{{"number(19,0)", false}, "numeric", -1},
	{{"number(20,0)", false}, "numeric", -1},
	{{"number(21,0)", false}, "numeric", -1},
	{{"number(22,0)", false}, "numeric", -1},
	{{"number(23,0)", false}, "numeric", -1},
	{{"number(24,0)", false}, "numeric", -1},
	{{"number(25,0)", false}, "numeric", -1},
	{{"number(26,0)", false}, "numeric", -1},
	{{"number(27,0)", false}, "numeric", -1},
	{{"number(28,0)", false}, "numeric", -1},
	{{"number(29,0)", false}, "numeric", -1},
	{{"number(30,0)", false}, "numeric", -1},
	{{"number(31,0)", false}, "numeric", -1},
	{{"number(32,0)", false}, "numeric", -1},
	{{"number(33,0)", false}, "numeric", -1},
	{{"number(34,0)", false}, "numeric", -1},
	{{"number(35,0)", false}, "numeric", -1},
	{{"number(36,0)", false}, "numeric", -1},
	{{"number(37,0)", false}, "numeric", -1},
	{{"number(38,0)", false}, "numeric", -1},
	{{"number", false}, "numeric", -1},
	{{"numeric", false}, "numeric", -1},
	{{"date", false}, "timestamp", -1},
	{{"long", false}, "text", -1},
	{{"interval day to second", false}, "interval day to second", -1},
	{{"interval year to month", false}, "interval year to month", 0},
	{{"timestamp", false}, "timestamp", -1},
	{{"timestamp with local time zone", false}, "timestamptz", -1},
	{{"timestamp with time zone", false}, "timestamptz", -1},
	{{"date", false}, "date", -1},
	{{"char", false}, "char", -1},
	{{"nchar", false}, "char", -1},
	{{"nvarchar2", false}, "varchar", -1},
	{{"varchar", false}, "varchar", -1},
	{{"varchar2", false}, "varchar", -1},
	{{"long raw", false}, "bytea", 0},
	{{"raw", false}, "bytea", 0},
	{{"decimal", false}, "decimal", -1},
	{{"rowid", false}, "text", 0},
	{{"urowid", false}, "text", 0},
	{{"xmltype", false}, "text", 0},
	/* large objects */
	{{"bfile", false}, "text", 0},
	{{"blob", false}, "bytea", 0},
	{{"clob", false}, "text", 0},
	{{"nclob", false}, "text", 0}
};
```