# Component Architecture
## Component Diagram
![img](https://www.highgo.ca/wp-content/uploads/2025/04/synchdb-component-diag3.png)

Each SynchDB worker consists of components and modules as shown in the component diagram and listed below:

1. Event Fetcher
2. JVM + DBZ Initializer
3. Request Handler
4. JSON Parser
5. Object Mapping Engine
6. Stats Collector
7. DDL Converter
8. DML Converter
9. Error Handler
10. SPI Client
11. Executor API

### 1) Event Fetcher
The Event Fetcher is primarily responsible for fetching a batch of JSON change events from embedded Debezium Runner running inside Java Virtual Machine (JVM). This is done via Java Native Interface (JNI) library by periodically calling a JAVA method that returns a JAVA `List` of `String`, which represents each change request in JSON. This `List` represents a `Batch` of JSON change events. JNI library is invoked again to iterate over this `List`, cast the contents from JAVA `String` to C string and send it to `4) JSON Parser` for further processing. The frequency of fetch is configurable via [`synchdb.naptime`](https://docs.synchdb.com/user-guide/configuration/), and the maximum size of the batch can be configured via [`synchdb.dbz_batch_size`](https://docs.synchdb.com/user-guide/configuration/).

When a batch is completed, meaning all the change events inside have been processed, it will invoke `markBatchComplete()` JAVA method via JNI to indicate that this batch has been completed successfully. This would cause the Debezium Runner to commit and advance the offset. More about batch management can he found [here](https://docs.synchdb.com/architecture/batch_change_handling/).

### 2) JVM + Debezium (DBZ) Initializer
The JVM + DBZ Initializer is primarily responsible for instanitiate a new JVM environment and run Debezium Runner (`dbz-engine-x.x.x.jar`) in it. This .jar file is by default installed in $LIBDIR as returned by `pg_config`. You can specify an alternative path for Debezium Runner .jar file by setting the environment variable `DBZ_ENGINE_DIR`. SynchDB currently uses `JNI_VERSION_10` as the JNI version, which is compatible between JAVA v10 to v18. In the future, however, we may up the JNI version to get the latest improvements and benefits for JNI. The maximum heap memory allocated to JVM can be configured via [`synchdb.jvm_max_heap_size`](https://docs.synchdb.com/user-guide/configuration/). If set to zero, JVM will automatically allocate the ideal heap size. 

Please note that each SynchDB worker will initialize and run a JVM instance, so the more workers you run, the more heap memory will be required. You can dump the current heap and non-heap memory usage of JVM of a SynchDB worker by invoking `synchdb_log_jvm_meminfo('$connector_name')` function, a memory summary will be logged in the PostgreSQL log file.

### 3) Request Handler
The Request Handler is primarily responsible for checking and handling any incoming state change request from the SynchDB user. Examples of such state change requests include going from "SYNCING" to "PAUSED", from "PAUSED" to "UPDATE OFFSET" ...etc. Below is the state diagram of synchdb. More information can be found [here](https://docs.synchdb.com/user-guide/utility_functions/#state-management).

![img](https://www.highgo.ca/wp-content/uploads/2025/04/synchdb-state-diag.png)

### 4) JSON Parser
The JSON Parser is responsible for parsing the incoming JSON change event into C structures that SynchDB can work with. SynchDB relies on PostgreSQL's native JSONB parser for all the parsing and iteration needs. For a DML JSON event, it first parses the "schema" section (also referred as metadata in this documentation) of the JSON change evnet to learn about how Debezium represents the user data. This include each field's data type representation, scale of the value...etc. Then it parses the "payload" to obtain the "before" and "after" values. Lastly, it parses the "source" section to obtain the table name, the database name and the connector type in which the data comes from.

For DML JSON event, the same sections will be parsed but with different attributes expected under "payload". It parses "tableChanges" under "payload" to learn about the designated table's column names, types and other properties.

Based on the nature of operation, the produced C structure data is then sent to `DDL Converter` for DDL requests or `DML Converter` for DML requests for next stage of processing. Below are examples of DDL and DML "payload" from a JSON event:

DML payload:
```json
{
  "payload": {
    "before": null,
    "after": {
      "id": 3,
      "g": {
        "wkb": "AQMAAAABAAAABQAAAAAAAAAAAAAAAAAAAAAAFEAAAAAAAAAAQAAAAAAAABRAAAAAAAAAAEAAAAAAAAAcQAAAAAAAAAAAAAAAAAAAHEAAAAAAAAAAAAAAAAAAABRA",
        "srid": null
      },
      "h": null
    },
    "source": {
      "version": "2.6.2.Final",
      "connector": "mysql",
      "name": "synchdb-connector",
      "ts_ms": 1743631156000,
      "snapshot": "last",
      "db": "inventory",
      "sequence": null,
      "ts_us": 1743631156000000,
      "ts_ns": 1743631156000000000,
      "table": "geom",
      "server_id": 0,
      "gtid": null,
      "file": "mysql-bin.000009",
      "pos": 1026620577,
      "row": 0,
      "thread": null,
      "query": null
    },
    "op": "r",
    "ts_ms": 1743631156410,
    "ts_us": 1743631156410423,
    "ts_ns": 1743631156410423395,
    "transaction": null
  }
}
```

DDL payload:
```json
  "payload": {
    "source": {
      "version": "2.6.2.Final",
      "connector": "sqlserver",
      "name": "synchdb-connector",
      "ts_ms": 1728337635149,
      "snapshot": "true",
      "db": "testDB",
      "sequence": null,
      "ts_us": 1728337635149000,
      "ts_ns": 1728337635149000000,
      "schema": "dbo",
      "table": "customers",
      "change_lsn": null,
      "commit_lsn": "00000195:000010a0:0003",
      "event_serial_no": null
    },
    "ts_ms": 1728337635150,
    "databaseName": "testDB",
    "schemaName": "dbo",
    "ddl": null,
    "tableChanges": [
      {
        "type": "CREATE",
        "id": "\"testDB\".\"dbo\".\"customers\"",
        "table": {
          "defaultCharsetName": null,
          "primaryKeyColumnNames": [
            "id"
          ],
          "columns": [
            {
              "name": "id",
              "jdbcType": 4,
              "nativeType": null,
              "typeName": "int identity",
              "typeExpression": "int identity",
              "charsetName": null,
              "length": 10,
              "scale": 0,
              "position": 1,
              "optional": false,
              "autoIncremented": true,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            },
            {
              "name": "first_name",
              "jdbcType": 12,
              "nativeType": null,
              "typeName": "varchar",
              "typeExpression": "varchar",
              "charsetName": null,
              "length": 255,
              "scale": null,
              "position": 2,
              "optional": false,
              "autoIncremented": false,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            },
            {
              "name": "last_name",
              "jdbcType": 12,
              "nativeType": null,
              "typeName": "varchar",
              "typeExpression": "varchar",
              "charsetName": null,
              "length": 255,
              "scale": null,
              "position": 3,
              "optional": false,
              "autoIncremented": false,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            },
            {
              "name": "email",
              "jdbcType": 12,
              "nativeType": null,
              "typeName": "varchar",
              "typeExpression": "varchar",
              "charsetName": null,
              "length": 255,
              "scale": null,
              "position": 4,
              "optional": false,
              "autoIncremented": false,
              "generated": false,
              "comment": null,
              "defaultValueExpression": null,
              "enumValues": null
            }
          ],
          "comment": null
        }
      }
    ]
  }
```

### 5) Object Mapping Engine
The Object Mapping Engine is responsible for loading and maintaining object mapping information under each active connector. These mapping information tells SynchDB how to map a source object to a destination object during DDL and DML processing. By default, Synchdb has no object mapping rules, it will use the default mapping rules to process the data.

An object could refer to a:
* table name.
* column name.
* data type.
* transform expression.

It is possible to map a source table name, column name and data type to a different destination table name, column name an a data type before mapping rules can be created using`synchdb_add_objmap()` function and all rules can be viewed by quering the `synchdb_objmap` table. More on object mapping [here](https://docs.synchdb.com/user-guide/object_mapping_rules/). A summary of what gets mapped to what can be viewed under `synchdb_att_view()` VIEW.


The `transform expression` is a SQL expression that will be run (if specified) after the data conversion is finished and before data is applied. This expression can be any expressions runnable in PostgreSQL, such as invoking another SQL function, or using operators. More information on object mapping rule can be found [here](https://docs.synchdb.com/user-guide/object_mapping_rules/).


### 6) Stats Collector
The Stats Collector is responsible for collecting statistic information about SynchDB's data processing since the beginning of the operation. This includes the number of DDLs and DMLs, how many CREATE, INSERT, UPDATE, DELETE operations have been processed, average batch size processed and several timestamps that describe the time when the data is first generated in the source, the time when the data is processed by Debezium and the time when the data is applied in PostgreSQL. These metrics can help user understand the processing behavior of SynchDB to tune and optimize settings to increase the processing performance. More on stats can be found [here](https://docs.synchdb.com/user-guide/stats).

### 7) DDL Converter
The DDL Converter is responsible for converting the DDL data produced by the "JSON Parser" to a format that can be understood by PostgreSQL. For DDLs, SynchDB relies on PostgreSQL SPI engine to process, so the output of the conversion is a normal SQL query string. DDL Converter examines the DDL data and has to work with "Object Mapping Engine" to correctly transform the table, column name or data type mappings between the source and destination. 

If a remote table named "employee" is to be mapped as "staff" in the destination according to "Object Mapping Engine", DDL Converter is responsible for resolving these name mappings and create the SQL query for SPI accordingly.

The converter currently can handle these DDL operations:

* CREATE TABLE
* DROP TABLE
* ALTER TABLE ALTER COLUMN
* ALTER TABLE ADD COLUMN
* ALTER TABLE DROP COLUMN

For CREATE and DROP, the converter is able to create a corresponding query string for SPI from the input DDL data. For ALTER, ADD and DROP COLUMN, the convert requires a visit to PostgreSQL catalog to learn about the existing table properties and will determine if there is a column to be added, dropped or altered. Debezium's JSON change event always contains the entire table's information and does not explicitly indicate what has been dropped or added. Therefore, the DDL Converter component is required to figure this information out and produce a correct query string. More on DDL replication can be found [here](https://docs.synchdb.com/user-guide/ddl_replication/)


### 8) DML Converter
The DML Converter is responsible for converting the DML data produced by the "JSON Parser" to a format that can be understood by PostgreSQL. For DMLs, SynchDB relies on PostgreSQL's executor APIs to directly apply the data to PostgreSQL, so the output of the conversion is in TupleTableSlot (TTS) format in which PostgreSQL executor understands. To produce the correct TTS for PostgreSQL, DML Converter relies on:

* DBZ metadata that describes how the payload data is represented
* PostgreSQL catalog (pg_class and pg_type) to learn about the table's information, each column's data type and properties. 
* Object Mapping Rules to determine if it needs to run additional transform expression on the processed data
* The payload data itself to process

DML Converter consists of several routines that can handle a particular input data type and produce a particular output type. Selecting the right routine for a particular conversion scenario could be a challenge because some data types may be user-defined or created by another extensions that SynchDB does not know much about. SynchDB has to be designed to handle both native and non-native data type that could exist in PostgreSQL.

The routine selection starts by looking at the data type created at the PostgreSQL, which can be divided into 2 types, each with slightly different handling techniques:

* native data types.
* non-native data types.

#### Handling Native Data Types

The image below shows the list of supported native data types and how SynchDB groups them together based on their nature (or category). For example, the numeric group contains all integer or float data types that are numeric in nature. They will give error if the data contains non-numeric characters. Likewise, different groups of data types requires a sepcific format of data in order to apply.

![img](https://www.highgo.ca/wp-content/uploads/2025/04/synchdb-native-types5.png)

Now that DML Converter knows how to produce the data for these supported native data types on the PostgreSQL side, it then looks at the DBZ metadata to learn how the source data is represented. This is needed because Debezium engine may encode the payload data to pack more information that requires decoding prior to processing the data, or use a structure to represent complex data types like Geometry. Without knowing how Debezium represents the data, the data processing is likely to produce undesired results, causing PostgreSQL to error during apply. Below is the list of formatting types that Debezium could represent a payload data with:

![img](https://www.highgo.ca/wp-content/uploads/2025/04/synchdb-dbztype.png)

With these 2 pieces of information, DML Converter knows what the input looks like and what the output should look like. It will select the best handler from its function matrix to process the data. For example, if destination type is `FLOAT4`, and source data type is formatted as `DBZTYPE_BYTES`, the function `handle_base64_to_numeric()` will be selected to process the data. The selected function is responsible for decode the binary input and compute it as a numeric. 


#### Handling Non-Native Data Types
It is possible that a table contains a column data type that is custom created by the user or created by another installed extension. In this case, the native data type handling mentioned above will not work becasue the type is not listed in the list of supported native data types. Instead, the DML Converter accesses the catalog, obtains the OID of the non-native data type, and looks up its "category" as defined in PostgreSQL. Below is a list of category supported by PostgreSQL as of version 17:

```
#define  TYPCATEGORY_INVALID	'\0'
#define  TYPCATEGORY_ARRAY		'A'
#define  TYPCATEGORY_BOOLEAN	'B'
#define  TYPCATEGORY_COMPOSITE	'C'
#define  TYPCATEGORY_DATETIME	'D'
#define  TYPCATEGORY_ENUM		'E'
#define  TYPCATEGORY_GEOMETRIC	'G'
#define  TYPCATEGORY_NETWORK	'I'
#define  TYPCATEGORY_NUMERIC	'N'
#define  TYPCATEGORY_PSEUDOTYPE 'P'
#define  TYPCATEGORY_RANGE		'R'
#define  TYPCATEGORY_STRING		'S'
#define  TYPCATEGORY_TIMESPAN	'T'
#define  TYPCATEGORY_USER		'U'
#define  TYPCATEGORY_BITSTRING	'V'
#define  TYPCATEGORY_UNKNOWN	'X'

```

The category tells DML Converter about the nature of the data type (numeric? string? datetime? ...etc) to help the converter select the right routine to process. For most cases, using type category paired with the DBZ metadata that describes how the input data payload is formatted is sufficient to select the right routine to process the data. However, in some cases, it may not be sufficient. For example, custom DATE, TIME, TIMESTAMP date types could all be categorized under `TYPCATEGORY_DATETIME`, so the converter does not know if it is working with a DATE, TIME or TIMESTAMP as each would produce different time formats. Currently, the covnerter looks for certain keywords from the data type name to identify. In the future, we may expose this part to let the user tell the converter exactly which routine to use should there be an ambiguity. Another example would be `TYPCATEGORY_USER` and `TYPCATEGORY_GEOMETRIC` which does not clearly indicate the data format. For these categories, the converter currently does not perform any further processing as it simply leaves the data payload as is. PostgreSQL may or may not reject such unprocessed data. This is why the transform feature next is important to give the DML converter a final chance to correct its data payload.

#### Data Transformation
After the input data has been processed by the logics as described above, the converter will then check if the user has configured a `transform expression` that shall be applied to the processed data before applying to PostgreSQL. A transform expression could be any PostgreSQL expressions, commands, or SQL functions that could be run on a psql prompt. It uses the `%d` as a placeholder character that will be replaced with the processed data during the transformation. For example, a transform expression "'>>>>>' || '%d' || '<<<<<'" will prepend and append additional characters to the processed string data. 

So, if a non-native data type has category TYPCATEGORY_USER, DML Converter does not have a suitable routine to process this data and will leave it as is, we can define a transform expression to call a custom SQL function from where it knows how to properly handle the data and produce a suitable output. For example, the expression, "to_my_composite_type('%d')" will call a user-defined SQL function `to_my_composite_type` with the data as input. The expression must have a return value as it will be fed into PostgreSQL during apply.



### 9) Error Handler
The Error Handler is primarily responsible for handling any error that could arise from each stage of data synchronization. Format Converter supports several error handling strategies that can be configured via "synchdb.error_handling_strategy" parameters with possible options listed below:

#### **exit (default)**
This is the default error strategy, which causes the connector worker to exit when an error has occured. The batch that it is currently working on, which resulted in error will not be marked as completed and the change events that have been successfully completed in the same batch will not be committed. User is expected to check the error message as returned in `synchdb_state_view()` or the log file to resolve the error. When the connector worker is restarted, the connector will automatically retry the same batch that has failed previously.

#### **retry**
This strategy adds a `restart_time` of 5 second to the connector worker, which causes PostgreSQL's bgworker engine to automatically start the worker every 5 second should it has exited. This means that when an error occurs, the connector worker will still exit, but unlike the `exit` strategy above, it will automatically be restarted by bgworker engine, which will retry on the same batch that has failed. It will continue to exit and restart until the error has been resolved.

#### **skip**
As the name suggests, when in `skip` error strategy, any error that the connector worker encountered will not cause the worker to exit, the error messages, however, will still be written to the log or `synchdb_state_view()`, but the connector itself will ignore the error and move on to processing the next change event and even the next batch. 


### 10) SPI Client
the SPI Client component exists under the Replication Agent, which serves as a bridge between PostgreSQL core and SynchDB. It is responsible for establishing a connection to SPI server, start a transaction, obtain a snapshot and execute a given SQL query created by the `DDL Converter` and destroy the connection. For each query to process, the SPI connection is created and destroyed, which may seem inefficient. Since the SPI is only used during DDL, which is normally not very frequent, it should be fine in terms of performance.

### 11) Executor APIs
Also residing in the Replication Agent. This component is responsible for initialize a executor context, open the table, acquire proper locks, create TupleTableSlot (TTS) from the output of DML Converter, call the executor API to execute INSERT, UPDATE, DELETE operations and do resource cleanup. This is generally a much faster approach to do data operations than SPI because it does not need to parse an input query string likst SPI does.