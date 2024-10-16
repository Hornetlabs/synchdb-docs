# 自定义数据类型映射

## 自定义规则文件
对于每种支持的连接器类型，SynchDB 都有一个默认的内置数据类型转换规则，将异构数据类型映射到与 PostgreSQL 兼容的类型。这个默认规则可以通过在 `synchdb_add_conninfo()` 期间指定的自定义 JSON 格式规则文件部分覆盖。Synchdb 在 PostgreSQL 服务器运行的 $PGDATA 目录下搜索指定的规则文件。以下是规则文件的示例：

myrule.json
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

规则文件必须包含一个名为 `"translation_rules"` 的 JSON 数组，其中包含多个对象：
* "translate_from"：要从异构数据库转换的数据类型名称。
* "translate_from_autoinc"：指示 `"translate_from"` 中指定的数据类型是否标记为自动增量。
* "translate_to"：要转换到的 PostgreSQL 数据类型。请参阅下面支持的 PostgreSQL 数据类型列表。
* "translate_to_size"：指示是否应转换 `"translate_to"` 中指定的数据类型的大小。例如：
    * 0：移除任何长度说明符，因为转换后的 PostgreSQL 数据类型不需要长度说明符。例如：TEXT
    * -1：使用源异构数据库的任何长度说明符（如果有），并直接映射到转换后的数据类型。例如：VARCHAR。
    * 其他：输入任何其他值以强制转换后数据类型的大小。请谨慎使用。

## 支持的 PostgreSQL 数据类型
PostgreSQL 支持从原生到复杂的各种数据类型。然而，SyncDB 目前仅支持 PostgreSQL 数据类型的一个子集。更多数据类型支持正在计划中。这意味着在定义自定义数据类型映射规则文件时，我们需要确保 `translate_to` 数据类型在以下支持的 PostgreSQL 数据类型列表中：

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

## 下一步计划
数据类型转换规则目前适用于所有情况。在未来，我们将通过允许数据类型转换仅在特定表或列内发生，或在转换后自动对值应用函数或表达式，来实现更精细的数据类型映射控制。（待定）