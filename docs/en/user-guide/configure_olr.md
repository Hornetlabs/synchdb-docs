# Configure Openlog Replicator for Oracle

## **Overview**

In addition to LogMiner-based Oracle replication, **SynchDB** supports **Openlog Replicator (OLR)** as a data source for Debezium-based Oracle connectors (type = `oracle`). This integration enables low-latency, redo log–based change data capture from Oracle databases via a standalone replication server. 

Furthermore, Synchdb also supports `native` Openlog Replicator connector, which does not rely on Debezium. This is a OLR client written purely in C that eliminates the JNI overheads.

This guide details the configuration steps necessary to:

* associate an Oracle connector with an OLR endpoint 
* or create a native Openlog Replicator Connector

Both can be done using `synchdb_add_olr_conninfo()` and `synchdb_del_olr_conninfo()`.

## **Requirements**

- **Openlog Replicator Version**: `1.3.0` (verified compatibility for Debezium 2.7.x)
- Oracle instance with redo logs accessible to OLR
- Openlog Replicator must be configured and running
- An existing Oracle connector in SynchDB (created using `synchdb_add_conninfo()`)

Refer to this [external guide](https://highgo.atlassian.net/wiki/external/OTUzY2Q2OWFkNzUzNGVkM2EyZGIyMDE1YzVhMDdkNWE) for details on deploying Openlog Replicator via Docker.

## **`synchdb_add_conninfo()`**

To create a **Debezium-based** or **Native** Openlog Replicator, users first have to create the connector with `synchdb_add_conninfo()` but with different `connector type`. For example:

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
                            'olr');    -- 'olr' for native openlog replicator or 'oracle' for debezium based openlog replicator connector

```

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

This instructs SynchDB to stream changes for the connector `oracleconn` from the Openlog Replicator instance running at 10.55.13.17:7070, using the Oracle source identifier ORACLE. Call `synchdb_start_engine_bgw` to start this connector.

```sql
SELECT synchdb_add_olr_conninfo('oracleconn', '10.55.13.17', 7070, 'ORACLE');

```

## **synchdb_del_olr_conninfo**

Removes the OLR configuration for a specific connector, reverting it to use LogMiner.

**Signature:**

```sql
synchdb_del_olr_conninfo(conn_name TEXT)

```

**Example:**

This command disables the use of OLR for oracleconn. Starting the connector with `synchdb_start_engine_bgw` will fall back to the default logmining strategy.

```sql
SELECT synchdb_del_olr_conninfo('oracleconn');

```

## **Behavior Notes**

* When both LogMiner and OLR configurations exist, SynchDB defaults to using Openlog Replicator for change capture.
* Restarting the connector is required after modifying its OLR configuration.
* The Oracle instance and OLR must remain synchronized in SCN progression and resetlogs identity for consistency.
* If OLR connection info exists, and connector type is `oracle` then it will use Debezium based Openlog Replicator client to stream changes.
* If OLR connection info exists, and connector type is `olr` then it will use native Openlog Replicator client to stream changes.