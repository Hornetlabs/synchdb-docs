# Function Reference

## Connector Management

### synchdb_add_conninfo

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
| `destination database` | Target PostgreSQL database | ✓ | `'postgres'` | Must exist in PostgreSQL |
| `table` | Table specification pattern | ☐ | `'[db].[table]'` | Empty = replicate all tables |
| `connector` | Connector type (`mysql`/`sqlserver`) | ✓ | `'mysql'` | See supported connectors above |
| `rule file` | Data type translation rules | ☐ | `'myrule.json'` | Must be in $PGDATA directory |

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
    'mysql',        -- Connector type
    'myrule.json'   -- Rules file
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
    'sqlserver',
    'mssql_rules.json'
);
```

### Basic Control Functions

#### synchdb_start_engine_bgw
**Purpose**: Initiates a connector
```sql
SELECT synchdb_start_engine_bgw('mysqlconn');
```

#### synchdb_pause_engine
**Purpose**: Temporarily halts a running connector
```sql
SELECT synchdb_pause_engine_bgw('mysqlconn');
```

#### synchdb_resume_engine
**Purpose**: Resumes a paused connector
```sql
SELECT synchdb_resume_engine('mysqlconn');
```

#### synchdb_stop_engine_bgw
**Purpose**: Terminates a connector
```sql
SELECT synchdb_stop_engine('mysqlconn');
```

## State Management

### synchdb_state_view
**Purpose**: Monitors connector states and status

```sql
SELECT * FROM synchdb_state_view();
```

**Return Fields**:

| Field | Description | Type |
|-|-|-|
| `id` | Connector slot identifier | Integer |
| `connector` | Connector type (`mysql` or `sqlserver`) | Text |
| `conninfo_name` | Associated connector name | Text |
| `pid` | Worker process ID | Integer |
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
- ⚫ `unknown` - Indeterminate state

### synchdb_set_offset
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

## Snapshot Management

### synchdb_restart_connector
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

**Example**:
```sql
-- Restart with specific snapshot mode
SELECT synchdb_restart_connector('mysqlconn', 'initial');
```

---
📝 **Additional Notes**:

- Always validate connector configuration before starting
- Monitor system resources during snapshot operations
- Back up PostgreSQL destination database before major operations
- Test connectivity from PostgreSQL server to source database
- Ensure source database has required permissions configured
- Regular monitoring of error logs recommended