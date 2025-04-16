---
weight: 110
---
# Function Reference

This page documents all the SQL functions / views added by SynchDB.

## **Connector Management**

### **synchdb_add_conninfo**

**Purpose**: Creates a new connector configuration

**Parameters**:

| Parameter | Description | Required | Example | Notes |
|:-:|:-|:-:|:-|:-|
| `name` | Unique identifier for this connector | ✓ | `'mysqlconn'` | Must be unique across all connectors |
| `hostname` | IP/hostname of heterogeneous database | ✓ | `'127.0.0.1'` | Support IPv4, IPv6, and hostnames |
| `port` | Port number for database connection | ✓ | `3306` | Default: MySQL(3306), SQLServer(1433) |
| `username` | Authentication username | ✓ | `'mysqluser'` | Requires appropriate permissions |
| `password` | Authentication password | ✓ | `'mysqlpwd'` | Stored securely |
| `source database` | Source database name | ✓ | `'inventory'` | Must exist in source system |
| `destination database` | Target PostgreSQL database | ✓ | `'postgres'` | (deprecated) will always be adjusted to the same database where SynchDB is installed |
| `table` | Table specification pattern | ☐ | `'[db].[table]'` | Empty = replicate all tables, support regular expressions (for example, mydb.testtable*), use `file:` prefix to make connector read table list from a JSON file (for example, file:/path/to/filelist.json). See below for file format |
| `connector` | Connector type (`mysql`/`sqlserver`) | ✓ | `'mysql'` | See supported connectors above |

**Tablelist File Example**:
```json
{
    "table_list":
    [
        "mydb.table1",
        "mydb.table2",
        "mydb.table3",
        "mydb.table4"
    ]
}
```

**Example Usage**:
```sql
-- MySQL Example
SELECT synchdb_add_conninfo(
    'mysqlconn',    -- Connector name
    '127.0.0.1',    -- Host
    3306,           -- Port
    'mysqluser',    -- Username
    'mysqlpwd',     -- Password
    'inventory',    -- Source DB
    'postgres',     -- Target DB
    '',             -- Tables (empty for all)
    'mysql'         -- Connector type
);

-- SQL Server Example
SELECT synchdb_add_conninfo(
    'sqlserverconn',
    '127.0.0.1',
    1433,
    'sa',
    'MyPassword123',
    'testDB',
    'postgres',
    'dbo.orders',   -- Specific table
    'sqlserver'
);

-- Oracle Example
SELECT synchdb_add_conninfo(
    'oracleconn',
    '127.0.0.1',
    1521,
    'c##dbzuser',
    'dbz',
    'mydb',
    'postgres',
    '',   -- all tables
    'oracle'
);
```

### **synchdb_add_objmap**

**Purpose**: Adds an object mapping rule per connector

| Parameter | Description | Required | Example | Notes |
|:-:|:-|:-:|:-|:-|
| `name` | Unique identifier for this connector | ✓ | `'mysqlconn'` | Must be unique across all connectors |
| `object type` | type of object mapping | ✓ | `'table'` | can be `table` to map a table name, `column` to map a column name, `datatype` to map a data type, or `transform` to run a data transform expression |
| `source object` | source object represented as fully-qualified name | ✓ | `inventory.customers` | the object name as represented in the remote database |
| `destination object` | destination object name | ✓ | `'schema1.people'` | The destination object name in PostgreSQL side. Can be a fully-qualified table name, a column name, a data type or transform expression |

```sql
SELECT synchdb_add_objmap('mysqlconn','table','inventory.customers','schema1.people');
SELECT synchdb_add_objmap('mysqlconn','column','inventory.customers.email','contact');
SELECT synchdb_add_objmap('mysqlconn','datatype','point|false','text|0');
SELECT synchdb_add_objmap('mysqlconn','datatype','inventory.geom.g','geometry|0');
SELECT synchdb_add_objmap('mysqlconn','transform','inventory.products.name','''>>>>>'' || ''%d'' || ''<<<<<''');
```
**Ways to represent a `table` mapping:**
* `source object` represents the table in fully-qualified name in remote database
* `destination object` represents the table name in PostgreSQL. It can be just a name (default to public schema) or in schema.name format. 

**Ways to represent a `column` mapping:**
* `source object` represents the column in fully-qualified name in remote database
* `destination object` represents the column name in PostgreSQL. No need to format it as fully-qualified column name.

**Ways to represent a `datatype` mapping:**
* `source object` can be expressed as one of:
    * a fully-qualified column (inventory.geom.g). This means the data type mapping applies to this particular column only.
    * a general data type string (int). Use a pipe (|) to add if it is a autoincrement data type (int|true for autoincremented int) or (int|false for a non-autoincremented int). This means data type mapping applies to all data type with the matching condition.

* `destination object` should be expressed as a general data type string that exists in PostgreSQL. Use a pipe (|) to overwrite the size (text|0 to overwrite the size to 0 because text is variable size) or (varchar|-1 to use whatever size that comes with the change event)

**Ways to represent a `transform` mapping:**
* `source object` represents the column to be transformed
* `destination object` represents an expression to be run on the column data before it is applied to PostgreSQL. Use %d as a placeholder for input column data. In case of geometry type, use %w for WKB and %s for SRID. 

### **synchdb_add_extra_conninfo**

**Purpose**: Configures extra connector parameters to an existing connector created by `synchdb_add_conninfo`

| Parameter | Description | Required | Example | Notes |
|:-:|:-|:-:|:-|:-|
| `name` | Unique identifier for this connector | ✓ | `'mysqlconn'` | Must be unique across all connectors |
| `ssl_mode` | SSL mode | ☐ | `'verify_ca'` | can be one of: <br><ul><li> "disabled" - no SSL is used. </li><li> "preferred" - SSL is used if server supports it. </li><li> "required" - SSL must be used to establish a connection. </li><li> "verify_ca" - connector establishes TLS with the server and will also verify server's TLS certificate against configured truststore. </li><li> "verify_identity" - same behavior as verify_ca but it also checks the server certificate's common name to match the hostname of the system. |
| `ssl_keystore` | keystore path | ☐ | `/path/to/keystore` | path to the keystore file |
| `ssl_keystore_pass` | keystore password | ☐ | `'mykeystorepass'` | password to access the keystore file |
| `ssl_truststore` | trust store path | ☐ | `'/path/to/truststore'` | path to the truststore file |
| `ssl_truststore_pass` | trust store password | ☐ | `'mytruststorepass'` | password to access the truststore file |


```sql
SELECT synchdb_add_extra_conninfo('mysqlconn', 'verify_ca', '/path/to/keystore', 'mykeystorepass', '/path/to/truststore', 'mytruststorepass');
```

### **synchdb_del_extra_conninfo**

**Purpose**: Deletes extra connector paramters created by `synchdb_add_extra_conninfo`
```sql
SELECT synchdb_del_extra_conninfo('mysqlconn');
```

### **synchdb_del_conninfo**

**Purpose**: Deletes connector information created by `synchdb_add_conninfo`
```sql
SELECT synchdb_del_extra_conninfo('mysqlconn');
```

### **synchdb_del_objmap**

**Purpose**: Disables object mapping records created by `synchdb_add_objmap`

| Parameter | Description | Required | Example | Notes |
|:-:|:-|:-:|:-|:-|
| `name` | Unique identifier for this connector | ✓ | `'mysqlconn'` | Must be unique across all connectors |
| `object type` | type of object mapping | ✓ | `'table'` | can be `table` to map a table name, `column` to map a column name, `datatype` to map a data type, or `transform` to run a data transform expression |
| `source object` | source object represented as fully-qualified name | ✓ | `inventory.customers` | the object name as represented in the remote database |

```sql
SELECT synchdb_del_extra_conninfo('mysqlconn', 'transform', 'inventory.products.name');
```

## **Basic Control Functions**

### **synchdb_start_engine_bgw**

**Purpose**: Starts a connector
```sql
SELECT synchdb_start_engine_bgw('mysqlconn');
```
You may also include snapshot mode to start the connector with, otherwise the `initial` mode will be used by default. See below for list of different snapshot modes.
```sql
-- capture table schema and proceed to stream new changes
SELECT synchdb_start_engine_bgw('mysqlconn', 'no_data');

-- always re-capture table schema, existing data and proceed to stream new changes
SELECT synchdb_start_engine_bgw('mysqlconn', 'always');
```

### **synchdb_pause_engine**

**Purpose**: Temporarily halts a running connector
```sql
SELECT synchdb_pause_engine_bgw('mysqlconn');
```

### **synchdb_resume_engine**

**Purpose**: Resumes a paused connector
```sql
SELECT synchdb_resume_engine('mysqlconn');
```

### **synchdb_stop_engine_bgw**

**Purpose**: Terminates a connector
```sql
SELECT synchdb_stop_engine('mysqlconn');
```

### **synchdb_reload_objmap**

**Purpose**: Causes a connector to load object mapping rules again
```sql
SELECT synchdb_reload_objmap('mysqlconn');
```

## **State Management**

### **synchdb_state_view**

**Purpose**: Monitors connector states and status

```sql
SELECT * FROM synchdb_state_view();
```

**Return Fields**:

| Field | Description | Type |
|-|-|-|
| `name` | Associated connector name | Text |
| `connector_type` | Connector type (`mysql` or `sqlserver`) | Text |
| `pid` | Worker process ID | Integer |
| `stage` | Current connector stage | Text |
| `state` | Current connector state | Text |
| `err` | Latest error message | Text |
| `last_dbz_offset` | Last recorded Debezium offset | JSON |

**Possible States**:

- 🔴 `stopped` - Inactive
- 🟡 `initializing` - Starting up
- 🟠 `paused` - Temporarily halted
- 🟢 `syncing` - Actively polling
- 🔵 `parsing` - Processing events
- 🟣 `converting` - Transforming data
- ⚪ `executing` - Applying changes
- 🟤 `updating offset` - Updating checkpoint
- 🟨 `restarting` - Reinitializing
- ⚪ `dumping memory` - JVM is prepaaring to dump memory info in log file
- ⚫ `unknown` - Indeterminate state

**Possible Stages**:

- `initial snapshot` - connector is performing initial snapshot (building table schema and optionally the initial data)
- `change data capture` - connector is streaming subsequent table changes (CDC)
- `schema sync` - connector is copying table schema only

### **synchdb_stats_view**

**Purpose**: Collects connector processing statistics cumulatiely

```sql
SELECT * FROM synchdb_stats_view();
```

| Field | Description | Type |
|-|-|-|
| name | Associated connector name | Text |
| ddls | Number of DDLs operations completed | Bigint |
| dmls | Number of DMLs operations completed | Bigint |
| reads | Number of READ events completed during initial snapshot stage | Bigint |
| creates | Number of CREATES events completed during CDC stage | Bigint |
| updates | Number of UPDATES events completed during CDC stage | Bigint |
| deletes | Number of DELETES events completed during CDC stage | Bigint |
| bad_events | Number of bad events ignored (such as empty events, unsupported DDL events..etc) | Bigint |
| total_events | Total number of events processed (including bad_events) | Bigint |
| batches_done | Number of batches completed | Bigint |
| avg_batch_size | Average batch size (total_events / batches_done) | Bigint |

### **synchdb_reset_stats**

**Purpose**: Resets all statistic information of given connector name

```sql
SELECT synchdb_reset_stats('mysqlconn');
```

### **synchdb_set_offset**

**Purpose**: Configures custom start position

**Example for MySQL**:
```sql
SELECT synchdb_set_offset(
    'mysqlconn', 
    '{"ts_sec":1725644339,"file":"mysql-bin.000004","pos":138466,"row":1,"server_id":223344,"event":2}'
);
```

**Example for SQL Server**:
```sql
SELECT synchdb_set_offset(
    'sqlserverconn',
    '{"event_serial_no":1,"commit_lsn":"00000100:00000c00:0003","change_lsn":"00000100:00000c00:0002"}'
);
```

### **synchdb_log_jvm_meminfo**

**Purpose**: Cause the Java Virtual Machine (JVM) to log the current heap and non-heap usages statistics.
```sql
SELECT synchdb_log_jvm_meminfo('mysqlconn');
```

Check the PostgreSQL log file:
```
2024-12-09 14:34:21.910 PST [25491] LOG:  Requesting memdump for mysqlconn connector
2024-12-09 14:34:21 WARN  DebeziumRunner:297 - Heap Memory:
2024-12-09 14:34:21 WARN  DebeziumRunner:298 -   Used: 19272600 bytes
2024-12-09 14:34:21 WARN  DebeziumRunner:299 -   Committed: 67108864 bytes
2024-12-09 14:34:21 WARN  DebeziumRunner:300 -   Max: 2147483648 bytes
2024-12-09 14:34:21 WARN  DebeziumRunner:302 - Non-Heap Memory:
2024-12-09 14:34:21 WARN  DebeziumRunner:303 -   Used: 42198864 bytes
2024-12-09 14:34:21 WARN  DebeziumRunner:304 -   Committed: 45023232 bytes
2024-12-09 14:34:21 WARN  DebeziumRunner:305 -   Max: -1 bytes

```

### **synchdb_att_view**

**Purpose**: Displays a side-by-side view of a connector's data type, name mapping and transform rule relationships between foreign and local tables.

```sql
SELECT * FROM synchdb_att_view();
```

**Return Fields**:

| Field | Description | Type |
|-|-|-|
| `name` | Connector identifier | Text |
| `attnum` | Attribute number | Integer |
| `ext_tbname` | table name as appeared remotely | Text |
| `pg_tbname` | mapped table name in PostgreSQL | Text |
| `ext_attname` | column name as appeared remotely | Text |
| `pg_attname` | mapped column name in PostgreSQL | Text |
| `ext_atttypename` | data type as appeared remotely | Text |
| `pg_atttypename` | mapped data type in PostgreSQL | Text |
| `transform` | transform expression | Text |

## **Snapshot Management**

### **synchdb_restart_connector**

**Purpose**: Reinitializes connector with specified snapshot mode

**Snapshot Modes**:

| Mode | Description | Use Case |
|:-:|-|-|
| `always` | Full snapshot on every start | Complete data verification |
| `initial` | First-time snapshot only | Normal operations |
| `initial_only` | One-time snapshot, then stop | Data migration |
| `no_data` | Structure only, no data | Schema synchronization |
| `never` | Skip snapshot, stream only | Real-time updates |
| `recovery` | Rebuilds from source | Disaster recovery |
| `when_needed` | Conditional snapshot | Automatic recovery |
| `schemasync` | Structure only, no data, no CDC | normal operations |

**Example**:
```sql
-- Restart with specific snapshot mode
SELECT synchdb_restart_connector('mysqlconn', 'initial');

-- Start with specific snapshot mode
SELECT synchdb_start_engine_bgw('mysqlconn', 'always');
```

---
📝 **Additional Notes**:

- Always validate connector configuration before starting
- Monitor system resources during snapshot operations
- Back up PostgreSQL destination database before major operations
- Test connectivity from PostgreSQL server to source database
- Ensure source database has required permissions configured
- Regular monitoring of error logs recommended