# Custom Data Type Mapping

## Custom Rule File
For each supported connector type, SynchDB has a default, built-in data type translation rule that maps heterogeneous data types to those compatible in PostgreSQL. This default rule can be partially overwritten with a custom JSON formatted rule file that is specified during `synchdb_add_conninfo()`. Synchdb seraches for specified rule file under $PGDATA directory where PostgreSQL server runs on. Here's an example of rule file:

myrule.json
```
{
    "translation_rules":
    [
        {
            "translate_from": "LINESTRING",
            "translate_from_autoinc": false,
            "translate_to": "TEXT",
            "translate_to_size": 0
        },
        {
            "translate_from": "BIGINT UNSIGNED",
            "translate_from_autoinc": false,
            "translate_to": "NUMERIC",
            "translate_to_size": -1
        }
    ]
}
```

The rule file must contain a JSON array with name `"translation_rules"` that contains multiple objects:
* "translate_from": the data type name from heterogeneous database to translate from.
* "translate_from_autoinc": indicate if the data type specified in `"translate_from"` is marked as auto increment.
* "translate_to": the PostgreSQL data type to translate to. See below for list of supported PostgreSQL data types.
* "translate_to_size": indicate if we should transform the size of the data type specifed in `"translate_to"`. For example:
    * 0: Remove any length specifier because the PostgreSQL data type translated to does not need length specifier. For example: TEXT
    * -1: Use whatever length specifier from the source heterogeneous database (if available) and map it directly to the translated datatype. For example: VARCHAR.
    * others: put any other values to force the size of translated data type. Use it with caution.

## Supported PostgreSQL Data Type
PostgreSQL supports a range of data types from native to complex data types. However, SyncDB only supports a subset of PostgreSQL data types as of now. More data type support is in the plan. This means that when defining a custom data type mapping rule file, we need to ensure the `translate_to` data type falls within the list of supported PostgreSQL data type below:

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


## Next Steps
The data type translation rule currently applies to all. In the future, we will have more granular control of the data type mapping by allowing a data type translation to happen only within specific tables or columns. Or automatically apply a function or expression to the value after the translation. (To be Determined)
