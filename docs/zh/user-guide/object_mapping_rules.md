---
weight: 60
---

# 对象映射规则

SynchDB 具有默认名称和数据类型映射规则来处理传入的更改事件。在大多数情况下，默认规则都可以正常工作。但是，如果您有特定的转换要求，或者默认规则不适合您，则可以将自己的对象映射规则配置到特定连接器。请按照以下工作流程查看和调整任何特定的对象映射规则。

## 创建连接器并以 `schemasync` 模式启动它

`schemasync` 是一种特殊模式，它使连接器连接到远程数据库并尝试仅同步指定表的架构。完成此操作后，连接器将处于 `暂停` 状态，用户可以查看使用默认规则创建的所有表和数据类型，并在需要时进行更改。

```sql
SELECT synchdb_add_conninfo(
    'mysqlconn',
    '127.0.0.1',
    3306,
    'mysqluser',
    'mysqlpwd',
    'inventory',
    'postgres',
    '',
    'mysql');

SELECT synchdb_start_engine_bgw('mysqlconn', 'schemasync');
```

## 确保连接器处于暂停状态

```sql
SELECT name, connector_type, pid, stage, state FROM synchdb_state_view;
     name      | connector_type |  pid   |    stage    |  state
---------------+----------------+--------+-------------+---------
 mysqlconn     | mysql          | 579845 | schema sync | polling

```

## 查看默认创建的表的映射规则

```sql
postgres=# select * from synchdb_att_view;
   name    | type  | attnum |         ext_tbname         |         pg_tbname          | ext_attname | pg_attname  | ext_atttypename | pg_atttypename | transform
-----------+-------+--------+----------------------------+----------------------------+-------------+-------------+-----------------+----------------+-----------
 mysqlconn | mysql |      1 | inventory.addresses        | inventory.addresses        | id          | id          | INT             | int4           |
 mysqlconn | mysql |      2 | inventory.addresses        | inventory.addresses        | customer_id | customer_id | INT             | int4           |
 mysqlconn | mysql |      3 | inventory.addresses        | inventory.addresses        | street      | street      | VARCHAR         | varchar        |
 mysqlconn | mysql |      4 | inventory.addresses        | inventory.addresses        | city        | city        | VARCHAR         | varchar        |
 mysqlconn | mysql |      5 | inventory.addresses        | inventory.addresses        | state       | state       | VARCHAR         | varchar        |
 mysqlconn | mysql |      6 | inventory.addresses        | inventory.addresses        | zip         | zip         | VARCHAR         | varchar        |
 mysqlconn | mysql |      7 | inventory.addresses        | inventory.addresses        | type        | type        | ENUM            | text           |
 mysqlconn | mysql |      1 | inventory.customers        | inventory.customers        | id          | id          | INT             | int4           |
 mysqlconn | mysql |      2 | inventory.customers        | inventory.customers        | first_name  | first_name  | VARCHAR         | varchar        |
 mysqlconn | mysql |      3 | inventory.customers        | inventory.customers        | last_name   | last_name   | VARCHAR         | varchar        |
 mysqlconn | mysql |      4 | inventory.customers        | inventory.customers        | email       | email       | VARCHAR         | varchar        |
 mysqlconn | mysql |      1 | inventory.geom             | inventory.geom             | id          | id          | INT             | int4           |
 mysqlconn | mysql |      2 | inventory.geom             | inventory.geom             | g           | g           | GEOMETRY        | text           |
 mysqlconn | mysql |      3 | inventory.geom             | inventory.geom             | h           | h           | GEOMETRY        | text           |
 mysqlconn | mysql |      1 | inventory.products         | inventory.products         | id          | id          | INT             | int4           |
 mysqlconn | mysql |      2 | inventory.products         | inventory.products         | name        | name        | VARCHAR         | varchar        |
 mysqlconn | mysql |      3 | inventory.products         | inventory.products         | description | description | VARCHAR         | varchar        |
 mysqlconn | mysql |      4 | inventory.products         | inventory.products         | weight      | weight      | FLOAT           | float4         |
 mysqlconn | mysql |      1 | inventory.products_on_hand | inventory.products_on_hand | product_id  | product_id  | INT             | int4           |
 mysqlconn | mysql |      2 | inventory.products_on_hand | inventory.products_on_hand | quantity    | quantity    | INT             | int4           |
(20 rows)

```

## 定义自定义映射规则

用户可以使用`synchdb_add_objmap`函数创建自定义映射规则。它可用于映射表名、列名、数据类型并定义数据转换表达式规则

```sql
SELECT synchdb_add_objmap('mysqlconn','table','inventory.products','stuff');
SELECT synchdb_add_objmap('mysqlconn','table','inventory.customers','schema1.people');
SELECT synchdb_add_objmap('mysqlconn','column','inventory.customers.last_name','family_name');
SELECT synchdb_add_objmap('mysqlconn','column','inventory.customers.email','contact');
SELECT synchdb_add_objmap('mysqlconn','datatype','inventory.geom.g','geometry|0');
SELECT synchdb_add_objmap('mysqlconn','datatype','inventory.orders.quantity','bigint|0');
SELECT synchdb_add_objmap('mysqlconn','transform','inventory.products.name','''>>>>>'' || ''%d'' || ''<<<<<''');
```

## 审查迄今为止创建的所有对象映射规则

```sql
postgres=# select * from synchdb_objmap;
   name    |  objtype  | enabled |            srcobj             |           dstobj
-----------+-----------+---------+-------------------------------+----------------------------
 mysqlconn | table     | t       | inventory.products            | stuff
 mysqlconn | column    | t       | inventory.customers.last_name | family_name
 mysqlconn | column    | t       | inventory.customers.email     | contact
 mysqlconn | table     | t       | inventory.customers           | schema1.people
 mysqlconn | transform | t       | inventory.products.name       | '>>>>>' || '%d' || '<<<<<'
 mysqlconn | datatype  | t       | inventory.geom.g              | geometry|0
 mysqlconn | datatype  | t       | inventory.orders.quantity     | bigint|0
(7 rows)

```

## 重新加载对象映射规则

一旦定义了所有自定义规则，我们就需要向连接器发出信号来加载它们。这将导致连接器读取并应用对象映射规则。如果它发现当前 PostgreSQL 值与对象映射值之间存在差异，它将尝试更正映射。

```sql
SELECT synchdb_reload_objmap('mysqlconn');

```

## 再次检查 `synchdb_att_view` 是否有变化

```sql
SELECT * from synchdb_att_view;
   name    | type  | attnum |         ext_tbname         |         pg_tbname          | ext_attname | pg_attname  | ext_atttypename | pg_atttypename |         transform
-----------+-------+--------+----------------------------+----------------------------+-------------+-------------+-----------------+----------------+----------------------------
 mysqlconn | mysql |      1 | inventory.addresses        | inventory.addresses        | id          | id          | INT             | int4           |
 mysqlconn | mysql |      2 | inventory.addresses        | inventory.addresses        | customer_id | customer_id | INT             | int4           |
 mysqlconn | mysql |      3 | inventory.addresses        | inventory.addresses        | street      | street      | VARCHAR         | varchar        |
 mysqlconn | mysql |      4 | inventory.addresses        | inventory.addresses        | city        | city        | VARCHAR         | varchar        |
 mysqlconn | mysql |      5 | inventory.addresses        | inventory.addresses        | state       | state       | VARCHAR         | varchar        |
 mysqlconn | mysql |      6 | inventory.addresses        | inventory.addresses        | zip         | zip         | VARCHAR         | varchar        |
 mysqlconn | mysql |      7 | inventory.addresses        | inventory.addresses        | type        | type        | ENUM            | text           |
 mysqlconn | mysql |      1 | inventory.customers        | schema1.people             | id          | id          | INT             | int4           |
 mysqlconn | mysql |      2 | inventory.customers        | schema1.people             | first_name  | first_name  | VARCHAR         | varchar        |
 mysqlconn | mysql |      3 | inventory.customers        | schema1.people             | last_name   | family_name | VARCHAR         | varchar        |
 mysqlconn | mysql |      4 | inventory.customers        | schema1.people             | email       | contact     | VARCHAR         | varchar        |
 mysqlconn | mysql |      1 | inventory.geom             | inventory.geom             | id          | id          | INT             | int4           |
 mysqlconn | mysql |      2 | inventory.geom             | inventory.geom             | g           | g           | GEOMETRY        | geometry           |
 mysqlconn | mysql |      3 | inventory.geom             | inventory.geom             | h           | h           | GEOMETRY        | text           |
 mysqlconn | mysql |      1 | inventory.products         | public.stuff               | id          | id          | INT             | int4           |
 mysqlconn | mysql |      2 | inventory.products         | public.stuff               | name        | name        | VARCHAR         | varchar        | '>>>>>' || '%d' || '<<<<<'
 mysqlconn | mysql |      3 | inventory.products         | public.stuff               | description | description | VARCHAR         | varchar        |
 mysqlconn | mysql |      4 | inventory.products         | public.stuff               | weight      | weight      | FLOAT           | float4         |
 mysqlconn | mysql |      1 | inventory.products_on_hand | inventory.products_on_hand | product_id  | product_id  | INT             | int4           |
 mysqlconn | mysql |      2 | inventory.products_on_hand | inventory.products_on_hand | quantity    | quantity    | INT             | int8           |
```

## 恢复连接器或重做整个快照

一旦确认对象映射正确，我们就可以恢复连接器。请注意，恢复只会继续流式传输新的表更改。不会复制表的现有数据。
```sql
SELECT synchdb_resume_engine('mysqlconn');
```

要捕获表的现有数据，我们还可以使用新的对象映射规则重做整个快照：
```sql
SELECT synchdb_restart_connector('mysqlconn', 'always');
```











# 转换规则文件

转换规则文件是一个以 JSON 格式编写的附加配置文件，描述了 SynchDB 连接器在从远程异构数据库接收表数据时应遵循的多个转换规则。该文件应放置在 `$PGDATA` 目录下，并在使用 `synchdb_add_conninfo()` SQL 函数创建连接器时选择该文件。创建新连接器时可以不使用自定义转换规则文件，在这种情况下，将应用默认的转换规则。

## 示例规则文件
```json
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
  ],
  "ssl_rules":
  {
      "ssl_mode": "disabled",
      "ssl_keystore": null,
      "ssl_keystore_pass": null,
      "ssl_truststore": null,
      "ssl_truststore_pass": null
  }
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

## SSL 规则配置

如果需要 SSL 来建立与远程数据库的连接，则此部分是必需的。

| | 字段| 描述 |
|-|-|-|
| ssl_mode | 可以是以下之一：<br><ul><li> “disabled” - 不使用 SSL。</li><li> “preferred” - 如果服务器支持，则使用 SSL。</li><li> “required” - 必须使用 SSL 来建立连接。</li><li> “verify_ca” - 连接器与服务器建立 TLS，还将根据配置的信任库验证服务器的 TLS 证书。</li><li> “verify_identity” - 与 verify_ca 相同的行为，但它还会检查服务器证书的通用名称以匹配系统的主机名。|
| ssl_keystore | 密钥库文件的路径 |
| ssl_keystore_pass | 访问密钥库文件的密码 |
| ssl_truststore | 信任库文件的路径 |
| ssl_truststore_pass |访问信任库文件的密码 |