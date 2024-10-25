---
weight: 60
---
# 转换规则文件

转换规则文件是一个以 JSON 格式编写的附加配置文件，描述了 SynchDB 连接器在从远程异构数据库接收表数据时应遵循的多个转换规则。该文件应放置在 `$PGDATA` 目录下，并在使用 `synchdb_add_conninfo()` SQL 函数创建连接器时选择该文件。创建新连接器时可以不使用自定义转换规则文件，在这种情况下，将应用默认的转换规则。

## 示例规则文件
```
{
  "transform_datatype_rules": [
    {
      "translate_from": "GEOMETRY",
      "translate_from_autoinc": false,
      "translate_to": "TEXT",
      "translate_to_size": -1
    },
    {
      "translate_from": "POINT",
      "translate_from_autoinc": false,
      "translate_to": "TEXT",
      "translate_to_size": -1
    },
    {
      "translate_from": "inventory.geom.g.GEOMETRY",
      "translate_from_autoinc": false,
      "translate_to": "GEOMETRY",
      "translate_to_size": 0
    },
    {
      "translate_from": "inventory.orders.quantity.INT",
      "translate_from_autoinc": false,
      "translate_to": "BIGINT",
      "translate_to_size": 0
    }
  ],
  "transform_objectname_rules": [
    {
      "object_type": "table",
      "source_object": "inventory.orders",
      "destination_object": "schema1.orders"
    },
    {
      "object_type": "table",
      "source_object": "inventory.products",
      "destination_object": "products"
    },
    {
      "object_type": "column",
      "source_object": "inventory.orders.order_number",
      "destination_object": "ididid"
    },
    {
      "object_type": "column",
      "source_object": "inventory.orders.purchaser",
      "destination_object": "the_dude"
    },
    {
      "object_type": "column",
      "source_object": "inventory.orders.quantity",
      "destination_object": "the_numba"
    },
    {
      "object_type": "column",
      "source_object": "testDB.dbo.customers.first_name",
      "destination_object": "the_awesome_first_name"
    }
  ],
  "transform_expression_rules": [
    {
      "transform_from": "inventory.orders.quantity",
      "transform_expression": "case when %d < 500 then 0 else %d end"
    },
    {
      "transform_from": "inventory.geom.g",
      "transform_expression": "ST_SetSRID(ST_GeomFromWKB(decode('%w', 'base64')),%s)"
    },
    {
      "transform_from": "inventory.products.name",
      "transform_expression": "'>>>>>' || '%d' || '<<<<<'"
    },
    {
      "transform_from": "inventory.products.description",
      "transform_expression": "'>>>>>' || '%d' || '<<<<<'"
    }
  ]
}
```
## 转换数据类型规则

转换数据类型规则影响 SynchDB 如何将数据类型从源异构数据库映射到 PostgreSQL 的等效数据类型。可以编写这些规则以应用于所有源表或仅应用于选定的表。如果特定数据类型没有可用的转换规则，SynchDB 将使用 [默认数据类型映射](https://docs.synchdb.com/user-guide/default_datatype_mapping/) 规则。

自定义数据类型转换规则可以通过名为 `"transform_datatype_rules"` 的 JSON 数组定义，数组中的每个元素必须包含以下对象：

|                        	| 描述                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             	| 示例                                                                	|
|------------------------	|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------	|------------------------------------------------------------------------	|
| translate_from         	| 要转换的异构数据库中的数据类型名称或全限定名称 (FQN)。FQN 包含：[database].[schema（如果存在）].[table].[column].[data type]：<br><br><ul><li>如果指定了 FQN，则转换仅应用于特定表的特定列。</li><li>如果未使用 FQN，则只要数据类型名称匹配，转换将全局应用。</li></ul>                                                                                                  	| GEOMETRY<br>POINT<br>inventory.geom.g.GEOMETRY<br>inventory.orders.quantity.INT 	|
| translate_from_autoinc 	| 指示 `"translate_from"` 中指定的数据类型是否标记为自动递增。                                                                                                                                                                                                                                                                                                                                                                                                                  	| false<br>true                                                             	|
| translate_to           	| 要转换到的 PostgreSQL 数据类型。请参见下方支持的 PostgreSQL 数据类型列表。 <br><br>可以指定 PostgreSQL 不原生支持的数据类型，例如 GEOMETRY，请确保在使用连接器之前已安装该数据类型。                                                                                                                                                                                                     	| TEXT<br>VARCHAR<br>BIGINT<br>GEOMETRY                                           	|
| translate_to_size      	| 指示是否应该转换 `"translate_to"` 中指定的数据类型的大小：<br><br><ul><li> 0：删除长度说明符，因为转换后的 PostgreSQL 数据类型不需要长度说明符。例如：TEXT </li><li> -1：使用源异构数据库中的长度说明符（如果可用），并直接将其映射到转换后的数据类型。例如：VARCHAR(32)。</li><li> 其他：输入任何其他值以强制转换后的数据类型的大小。请谨慎使用。</li></ul> 	| 0<br>-1<br>45<br>7000                                                           	|

### 支持的 PostgreSQL 数据类型
 
SynchDB 支持以下 PostgreSQL 原生数据类型，可在创建表时进行转换。然而，仍然可以将外部数据类型转换为不在此列表中的 PostgreSQL 数据类型。例如，Postgis 添加的 `GEOMETRY` 数据类型。在这种情况下，表仍然会以 `GEOMETRY` 数据类型创建，但 SynchDB 接收到的数据将以 `TEXT` 格式化并插入。由你决定是否需要在应用到 PostgreSQL 之前通过表达式或 SQL 函数处理此数据（请参见下方的[转换表达式规则](#_5)）。

```sql
BOOLEAN (BOOLOID)
BIGINT (INT8OID)
SMALLINT (INT2OID)
INT (INT4OID)
INTEGER (INT4OID)
DOUBLE PRECISION (FLOAT8OID)
REAL (FLOAT4OID)
MONEY (MONEYOID)
NUMERIC (NUMERICOID)
CHAR (BPCHAROID)
CHARACTER (BPCHAROID)
TEXT (TEXTOID)
VARCHAR (VARCHAROID)
CHARACTER VARYING (VARCHAROID)
TIMESTAMPTZ (TIMESTAMPTZOID)
JSONB (JSONBOID)
UUID (UUIDOID)
VARBIT (VARBITOID)
BIT VARYING (VARBITOID)
BIT (BITOID)
DATE (DATEOID)
TIMESTAMP (TIMESTAMPOID)
TIME (TIMEOID)
BYTEA (BYTEAOID)
```
## 转换对象名称规则

转换对象名称规则影响 SynchDB 如何将源表或列名从异构数据库映射到 PostgreSQL 的表或列名。

可以通过名为 `"transform_objectname_rules"` 的 JSON 数组定义转换对象名称规则，数组中的每个元素必须包含以下对象：

| | 描述 | 示例 |
|-|-|-|
| object type        | 转换元素的对象类型。可以是： <br><br><ul><li> `table` 表示该转换适用于表名。 </li><li> `column` 表示该转换适用于列名。 </li></ul>| table<br>column |
| source_object      | 源对象的全限定名称 (FQN)，来自异构数据库。 <br><br><ul><li> 如果 `object_type` 是 `table`，FQN 包含 [database].[schema].[table] 或 [database].[table]。 </li><li> 如果 `object_type` 是 `column`，FQN 包含 [database].[schema].[table].[column] 或 [database].[table].[column]。</li></ul><br> 请注意，一些异构数据库（如 SQL Server）对名称是区分大小写的，因此请确保正确构造 FQN，保持正确的大小写。| mydb.myschema.mytable<br>yourdb.yourtable.yourcolumn |
| destination_object 	| 在 PostgreSQL 中用于表示异构数据库中 `source_object` 的名称： <br><br><ul><li>如果 `object_type` 是 `table`，`destination_object` 可以包含一个可选的架构名和表名，之间以点分隔。如果未指明架构，则使用默认的 `public` 架构。</li><li> 如果 `object_type` 是 `column`，`destination_object` 只能包含列名（不包括架构）。</li></ul>| myschema1.mytable1<br>mytable2<br>the_dude<br>the_numba |

如果表或列名没有匹配的转换对象名称规则，默认转换规则将自动应用，如下所示：

| 远程对象 FQN                  | PostgreSQL 端的默认对象名称转换规则|
|-|-|
| [database].[table]          | <ul><li> [database] → [schema]。 </li><li> [table] → [table]。</li></ul>|
| [database].[schema].[table] | <ul><li> [database] → [schema]。</li><li> [schema] 被忽略。 </li><li> [table] → [table]。</li></ul> |


## 转换表达式规则

转换表达式规则指示 SynchDB 是否需要在将接收到的数据应用于 PostgreSQL 之前执行额外的“表达式”操作。此功能允许 SynchDB 改变数据的表示方式，而无需在 PostgreSQL 端添加额外的应用逻辑。

可以通过名为 `"transform_expression_rules"` 的 JSON 数组定义转换表达式规则，数组中的每个元素必须包含以下对象：

| | 描述| 示例 |
|-|-|-|
| transform_from       | 远程列的全限定名称 (FQN)，可以是以下格式之一：<br><br><ul><li> [database].[schema].[table].[column] </li><li> [database].[table].[column] </li></ul>| inventory.orders.quantity<br>testDB.dbo.products.description|
| transform_expression 	| 要在接收到的数据上执行的表达式。可以使用以下占位符构造表达式：<ul><li> %d: 将被接收到的数据替换 </li><li> %w: 公认的二进制表示。当数据表示几何或地理数据时将存在此项 </li><li> %s: SRID。当数据表示几何或地理数据时将存在此项 </li></ul> 表达式可以用 PostgreSQL 支持的任何标准 SQL 语法编写。 | 1. ```case when %d < 500 then 0 else %d end``` <br>  如果值小于 500，则设置为 0，否则保留原始值<br><br> 2.```ST_SetSRID(ST_GeomFromWKB(decode('%w', 'base64')),%s)``` <br>*将 base64 编码的 Well-Known Binary (WKB) 几何数据转换为具有指定空间参考系统 (SRID) 的 PostGIS 几何对象* <br><br>3. ```'>>>>>' \|\| '%d' \|\| '<<<<<'``` <br>在值周围添加可视标记|
