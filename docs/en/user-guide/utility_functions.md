# Utility Function List

## synchdb_add_conninfo

Used to create a new connector information:

*  `name` - a unique identifier that represents this connection info

*  `hostname` - the IP address of heterogeneous database.

*  `port` - the port number to connect to.

*  `username` - user name to use to connect.

*  `password` - password to authenticate the username.

*  `source database` (optional) - the database name to replicate changes from. For example, the database that exists in MySQL. If empty, all databases from MySQL are replicated.

*  `destination` database - the database to apply changes to - For example, the database that exists in PostgreSQL. Must be a valid database that exists.

*  `table` (optional) - expressed in the form of `[database].[table]` that must exists in MySQL so the engine will only replicate the specified tables. If empty, all tables are replicated.

*  `connector` - the connector to use (MySQL, Oracle, or SQLServer). Currently only MySQL and SQLServer are supported.

*  `rule file` - a JSON-formatted rule file placed under $PGDATA that this connector shall apply to its default data type translation rules. See below for more detail.

Example:

```
SELECT synchdb_add_conninfo('mysqlconn','127.0.0.1',3306,'mysqluser', 'mysqlpwd', 'inventory', 'postgres', '', 'mysql', 'myrule.json');
```

## synchdb_start_engine_bgw
Used to start a connector:
* `name` - the name of connector to start

```
SELECT synchdb_start_engine_bgw('mysqlconn');
```

## synchdb_pause_engine
Used to pause a running connector:
* `name` - the name of connector to pause

```
SELECT synchdb_pause_engine_bgw('mysqlconn');
```

## synchdb_resume_engine
Used to resume a paused connector:
* `name` - the name of connector to resume

```
SELECT synchdb_resume_engine('mysqlconn');
```

## synchdb_stop_engine_bgw
Used to stop a paused or running connector:
* `name` - the name of connector to stop

```
SELECT synchdb_stop_engine('mysqlconn');
```

## synchdb_state_view
Used to examine all the running connectors and their states:
```
SELECT * FROM synchdb_state_view();
```

* id: unique identifier of a connector slot
* connector: the type of connector (mysql, oracle, sqlserver...etc)
* conninfo_name: the associated connection info name created by `synchdb_add_conninfo()`
* pid: the PID of the connector worker process
* state: the state of the connector. Possible states are:
    * stopped
    * initializing
    * paused
    * syncing
    * parsing
    * converting
    * executing
    * updating offset
    * unknown
* err: the last error message encountered by the worker which would have caused it to exit. This error could originated from PostgreSQL while processing a change, or originated from Debezium running engine while accessing data from heterogeneous database.
* last_dbz_offset: the last Debezium offset captured by synchdb. Note that this may not reflect the current and real-time offset value of the connector engine. Rather, this is shown as a checkpoint that we could restart from this offeet point if needed.

## synchdb_set_offset
Used to set a custom starting offset value:
* `name` - the name of connector to set a new offset
* `offset` - the offset value to set

```
SELECT synchdb_set_offset('mysqlconn', '{"ts_sec":1725644339,"file":"mysql-bin.000004","pos":138466,"row":1,"server_id":223344,"event":2}');
```
