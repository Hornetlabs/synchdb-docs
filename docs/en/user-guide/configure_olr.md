# Configure Openlog Replicator for Oracle

## **Overview**

In addition to LogMiner-based Oracle replication, **SynchDB** also supports **Openlog Replicator (OLR)** to stream from Oracle databases. There are 2 types of Openlog Replicator supported in SynchDB:

1. Debezium based Openlog Replicator
2. Native Openlog Replicator (not via Debezium) - at BETA

Both require Openlog Replicator to be configured according to the setup example below and connected to Oracle prior to streaming in SynchDB.

## **Requirements**

- **Openlog Replicator Version**: `1.3.0` (verified compatibility for Debezium 2.7.x)
- Oracle instance with redo logs accessible to OLR
- Additional permissions must be granted for OLR.  Refer to [here](https://docs.synchdb.com/getting-started/remote_database_setups/) for exact permission requirement.
- Openlog Replicator must be configured and running
- An existing Oracle connector in SynchDB (created using `synchdb_add_conninfo()`)

Refer to this [external guide](https://highgo.atlassian.net/wiki/external/OTUzY2Q2OWFkNzUzNGVkM2EyZGIyMDE1YzVhMDdkNWE) for details on deploying Openlog Replicator via Docker.

## **Openlog Replicator Configuration Example**

SynchDB's OLR support is built against the configuration example below. 

**Version 1.3.0**
```json
{
  "version": "1.3.0",
  "source": [
    {
      "alias": "SOURCE",
      "name": "ORACLE",
      "reader": {
        "type": "online",
        "user": "DBZUSER",
        "password": "dbz",
        "server": "//ora19c:1521/FREE"
      },
      "format": {
        "type": "json",
        "column": 2,
        "db": 3,
        "interval-dts": 9,
        "interval-ytm": 4,
        "message": 2,
        "rid": 1,
        "schema": 7,
        "timestamp-all": 1,
        "scn-all": 1
      },
      "memory": {
        "min-mb": 64,
        "max-mb": 1024
      },
      "filter": {
        "table": [
          {"owner": "DBZUSER", "table": ".*"}
        ]
      },
      "flags": 32
    }
  ],
  "target": [
    {
      "alias": "SYNCHDB",
      "source": "SOURCE",
      "writer": {
        "type": "network",
        "uri": "0.0.0.0:7070"
      }
    }
  ]
}

```

**Version 1.8.5**
```json
{
  "version": "1.8.5",
  "source": [
    {
      "alias": "SOURCE",
      "name": "ORACLE",
      "reader": {
        "type": "online",
        "user": "DBZUSER",
        "password": "dbz",
        "server": "//ora19c:1521/FREE"
      },
      "format": {
        "type": "json",
        "column": 2,
        "db": 3,
        "interval-dts": 9,
        "interval-ytm": 4,
        "message": 2,
        "rid": 1,
        "schema": 7,
        "timestamp-all": 1,
        "scn-type": 1
      },
      "memory": {
        "min-mb": 64,
        "max-mb": 1024,
        "swap-path": "/opt/OpenLogReplicator/olrswap"
      },
      "filter": {
        "table": [
          {"owner": "DBZUSER", "table": ".*"}
        ]
      },
      "flags": 32
    }
  ],
  "target": [
    {
      "alias": "DEBEZIUM",
      "source": "SOURCE",
      "writer": {
        "type": "network",
        "uri": "0.0.0.0:7070"
      }
    }
  ]
}

```

Please note the following:

- "source"."name": "ORACLE" -> this should match the `olr_source` value when defining OLR parameters via `synchdb_add_olr_conninfo()` (See below)
- "source"."reader"."user" -> this should match the `username` value when creating a connector via `synchdb_add_conninfo()`
- "source"."reader"."password" -> this should match the `password` value when creating a connector via `synchdb_add_conninfo()`
- "source"."reader"."server" -> this should contain the values of `hostname`, `port` and `source database` values when creating a connector via `synchdb_add_conninfo()`
- "source"."filter"."table":[] -> this filters the change events that Openlog Replicator captures. <<<**IMPORTANT**>>>: This is currently the only way to filter change events from Oracle as OLR implementations in SynchDB does not do any filtering at this moment. (The `table` and `snapshot table` values are ignored when creating a connector via `synchdb_add_conninfo()`) 
- "format":{} -> the specific paylod format ingested by Debezium based or native Openlog Replicator connector. Use these values as specified.
- "memory"."swap-path" -> this tells OLR where to write swap files in low memory scenario.
- "target".[0]."writer"."type": -> this must specify `network` as both Debezium and native Openlog Replicator connector communicate with Openlog Replicator via network
- "target".[0]."writer"."uri": -> this is the bind host and port Openlog Replicator listens on that SynchDB should be able to access via `olr_host` and `olr_port` when defining OLR parameters via `synchdb_add_olr_conninfo()`. 

## **`synchdb_add_conninfo()`**

To create a **Debezium-based** OLR connector (use `type` = 'oracle'):

```sql
SELECT synchdb_add_conninfo('olrconn',
                            'ora19c',
                            1521,
                            'DBZUSER',
                            'dbz',
                            'FREE',
                            'postgres',
                            'null',
                            'null',
                            'oracle');

```

To create a **Native** OLR connector (use `type` = 'olr'):

```sql
SELECT synchdb_add_conninfo('olrconn',
                            'ora19c',
                            1521,
                            'DBZUSER',
                            'dbz',
                            'FREE',
                            'postgres',
                            'null',
                            'null',
                            'olr');

```

<<<**IMPORTANT**>>> **SynchDB must be compiled and built with flag (WITH_OLR=1) to support native openlog replicator connector.**

More on creating a connector can be found [here](https://docs.synchdb.com/user-guide/create_a_connector/)

## **`synchdb_add_olr_conninfo()`**

With a connector created, the user then registers an Openlog Replicator endpoint for an existing Oracle connector.

**Signature:**

```sql
synchdb_add_olr_conninfo(
    conn_name TEXT,     -- Name of the connector
    olr_host TEXT,      -- Hostname or IP of the OLR instance
    olr_port INT,       -- Port number exposed by OLR (typically 7070)
    olr_source TEXT     -- Oracle source name as configured in OLR
)
```

**Example:**

This instructs SynchDB to stream changes for the connector `olrconn` from the Openlog Replicator instance running at olrhost:7070, using the Oracle source identifier ORACLE. Call `synchdb_start_engine_bgw` to start this connector.

```sql
SELECT synchdb_add_olr_conninfo('olrconn', 'olrhost', 7070, 'ORACLE');

```

## **synchdb_del_olr_conninfo**

Removes the OLR configuration for a specific connector, reverting it to use LogMiner.

**Signature:**

```sql
synchdb_del_olr_conninfo(conn_name TEXT)

```

**Example:**

This command disables the use of OLR for oracleconn. Starting the connector with `synchdb_start_engine_bgw` will fall back to the default logmining strategy (Debezium based OLR connector). If using native Openlog Replicator connector, the absense of OLR configuration will result in error at connector startup.

```sql
SELECT synchdb_del_olr_conninfo('olrconn');

```

## **Behavior Notes for Debezium based Openlog Replicator Connector**

* When both LogMiner and OLR configurations exist, SynchDB defaults to using Openlog Replicator for change capture.
* If OLR configurations are absent, SynchDB uses logmining strategy to stream changes.
* Restarting the connector is required after modifying its OLR configuration.

## **Behavior Notes for Native based Openlog Replicator Connector**

* Currently at BETA version.
* SynchDB manages connections to Openlog Replicator and streams changes without using Debezium.
* Requires OLR configuration or connector will error on startup.
* Relies on Debezium's oracle connector to complete initial snapshot and shuts down when done, subsequent CDC is done natively within SynchDB against Openlog Replicator.
* Relies on IvorySQL's oracle parser to handle DDL events. This must be compiled and installed prior to using native openlog replicator connector.
* Visit [here](https://github.com/bersler/OpenLogReplicator) for more information about Openlog Replicator.

