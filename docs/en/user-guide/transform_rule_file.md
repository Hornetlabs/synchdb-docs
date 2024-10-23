# Transform Rule File

Transform rule file is an additional configuration file written in JSON format that describes several transform rules that a SynchDB connector should follow when receiving a table data from remote heterogeneous database. This file shall be put under `$PGDATA` directory and is selected when a connector is created using `synchdb_add_conninfo()` SQL function. It is possible not to use a custom transform rule file when creating a new connector, in this case, default transfom rules will be applied.

## Sample rule file
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
  ]
}
```
## Transform Data Type Rules

Transform data type rules influence how SynchDB maps a data type from source heterogeneous database to an equivalent data type in PostgreSQL These rules can be written to apply to all source tables or only selected tables. If a data type rule is not available for a particular data type, SynchDB will use the [default data type mapping](https://docs.synchdb.com/user-guide/default_datatype_mapping/) rules instead. 

Custom data type transform rules can be defined with a JSON array with name `"transform_datatype_rules"` and each element in the array must contain the following objects:

|                        	| Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             	| Example                                                                	|
|------------------------	|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------	|------------------------------------------------------------------------	|
| **translate_from**         | the data type name or Fully Qualified Name (FQN) from heterogeneous database to transform from. The FQN consists of: [database].[schema (if present)].[table name].[column name].[data type]: <br><ul><li>If FQN is specified, then the transformation will only be applied to a specific table's specific column.</li><li> If FQN is not used, the transformation will be applied globally if data type name matches. </li></ul> | GEOMETRY<br>POINT<br>inventory.geom.g.GEOMETRY<br>inventory.orders.quantity.INT |
| **translate_from_autoinc** | indicate if the data type specified in `"translate_from"` is marked as auto increment.| false<br>true|
| **translate_to**           | the PostgreSQL data type to translate to. See below for list of supported PostgreSQL data types. <br>It is possible to specify data types not natively supported by PostgreSQL, for example GEOMETRY, please ensure that the data type has already been installed before starting a connector using it.| TEXT<br>VARCHAR<br>BIGINT<br>GEOMETRY|
| **translate_to_size**      | indicate if we should transform the size of the data type specifed in `"translate_to"`: <br><ul><li> 0: Remove any length specifier because the PostgreSQL data type transformed does not need length specifier. For example: TEXT </li><li> -1: Use whatever length specifier from the source heterogeneous database (if available) and map it directly to the translated datatype. For example: VARCHAR(32).</li><li> others: put any other values to force the size of translated data type. Use it with caution. </li></ul> 	| 0<br>-1<br>45<br>7000 |

### Supported PostgreSQL Data Types
 
SynchDB supports the following native PostgreSQL data types that can be transformed to during the table creation.  However, it is possible to transform a foreign data type to PostgreSQL data type not listed below. For example, the `GEOMETRY` data type added by Postgis. In this case, the table is still created with `GEOMETRY` data type, but the data that SynchDB receives will be formatted and inserted as `TEXT`. It is up to you to decide if this data needs to be processed by an expression or SQL function (see transform expression rules below) before it can be applied to PostgreSQL.

* BOOLEAN (BOOLOID)
* BIGINT (INT8OID)
* SMALLINT (INT2OID)
* INT (INT4OID)
* INTEGER (INT4OID)
* DOUBLE PRECISION (FLOAT8OID)
* REAL (FLOAT4OID)
* MONEY (MONEYOID)
* NUMERIC (NUMERICOID)
* CHAR (BPCHAROID)
* CHARACTER (BPCHAROID)
* TEXT (TEXTOID)
* VARCHAR (VARCHAROID)
* CHARACTER VARYING (VARCHAROID)
* TIMESTAMPTZ (TIMESTAMPTZOID)
* JSONB (JSONBOID)
* UUID (UUIDOID)
* VARBIT (VARBITOID)
* BIT VARYING (VARBITOID)
* BIT (BITOID)
* DATE (DATEOID)
* TIMESTAMP (TIMESTAMPOID)
* TIME (TIMEOID)
* BYTEA (BYTEAOID)

## Transform Object Name Rules

Transform object name rules influence how SynchDB maps a source table or column name from heterogeneous database to PostgreSQL table or column names.

Transform data type rules can be defined with a JSON array with name `"transform_objectname_rules" and each element in the array must contain the following objects:

|                    	| description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  	| example                                        	|
|--------------------	|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------	|------------------------------------------------	|
| **object type**        	| the object type of this transformation element. It can be: <br><ul><li> `table` to indicate the transformation is to be done on table names. </li><li>`column` to  indicate the transformation is to be done on column names. </li></ul>                                                                                                                                                                                                                                                                                  	| table<br>column                                   	|
| **source_object**      	| The Fully Qualified Name (FQN) of the source object from heterogeneous database. <br><ul><li> If `object_type` is `table`, the FQN consists of [database].[schema].[table] or [database].[table]. </li><li> If `object_type` is `column`, the FQN consists of [database].[shema].[table].[column] or [database].[table].[column]. </li></ul>  Please note that some heterogeneous databases (such as SQLServer) is case-sensitive with names, so please ensure the FQN is constructed correctly with the correct cases.  	| false<br>true                                     	|
| **destination_object** 	| The name to be used in PostgreSQL to represent the `source_object` from heterogeneous database: <br><ul><li>If `object_type` is `table`, the `destination_object` can contain an optional schema name followed by a table name separated by a dot. If no schema is indicated, the default `public` schema will be used.</li><li> if `object_type` is `column`, the `destination_object` can only contain a name (no schema).| myschema1.mytable1<br>mytable2<br>the_dude<br>the_numba |

If a table or column name does not have a matching transform object name rules, the default transform will be automatically applied as below:

| Remote object FQN           | Default object name transform rule on PostgreSQL side|
|-|-|
| [database].[table]          | <ul><li> [database] → [schema]. </li><li> [table] → [table]. </li></ul>|
| [database].[schema].[table] | <ul><li> [database] → [schema]. </li><li> [schema] is ignored. </li><li> [table] → [table]. </li></ul> |

## Transform Expression Rules

Transform expression rules indicate to SynchDB if it needs to perfrom additional "expressions" on the received data before applying them to PostgreSQL. This feature allows SynchDB to changes the representation of data without additional application logics on PostgreSQL site. 

Transform data type rules can be defined with a JSON array with name `"transform_expression_rules"` and each element in the array must contain the following objects:


| | Description | Example |
|-|-|-|
| **transform_from**      | the Fully Qualified Name (FQN) of a remote column that can be one of these formats:<br><ul><li> [database].[schema].[table].[column] </li><li> [database].[table].[column] </li></ul>| inventory.orders.quantity<br>testDB.dbo.products.description|
| **transform_expression** | the expression to execute on the received data. You can use these placeholder tokens to construct an expression: <br><ul><li> %d: will be swapped with the received data </li><li> %w: Well-known binary representation. Will be present if the data represents a geometry or geography data </li><li> %s: SRID. Will be present if the data represents a geometry or geography data </li></ul> <br> the expression can be written in any standard SQL syntax supported by PostgreSQL. | 1. ```case when %d < 500 then 0 else %d end``` <br> sets a value to 0 if it's less than 500, otherwise keeps its original value <br><br> 2.```ST_SetSRID(ST_GeomFromWKB(decode('%w', 'base64')),%s)``` <br>*Converts base64-encoded Well-Known Binary (WKB) geometry data to a PostGIS geometry object with a specified spatial reference system (SRID)* <br><br>3. ```'>>>>>' \|\| '%d' \|\| '<<<<<'``` <br>adds visual markers around a value|

