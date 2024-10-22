# Utility Function List

## synchdb_add_conninfo

Used to create a new connector information:

|        argumet        |                                                                                                                description                                                                                                                |
|:--------------------: |:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
| name                  | a unique identifier that represents this connector info                                                                                                                                                                                   |
| hostname              | the IP address or hostname of the heterogeneous database.                                                                                                                                                                                 |
| port                  | the port number to connect to the heterogeneous database.                                                                                                                                                                                 |
| username              | user name to use to authenticate with heterogeneous database.                                                                                                                                                                             |
| password              | password to authenticate the username                                                                                                                                                                                                     |
| source database       | this is the name of source database in heterogeneous database that we want to replicate changes from.                                                                                                                                     |
| destination database  | this is the name of destination database in PostgreSQL to apply changes to. It must be a valid database that exists in PostgreSQL.                                                                                                        |
| table                 | (optional) - expressed in the form of `[database].[table]` or `[database].[schema].[table]` that must exists in heterogeneous database so the engine will only replicate the specified tables. If left empty, all tables are replicated.  |
| connector             | the connector type to use (MySQL, Oracle, SQLServer... etc).                                                                                                                                                                              |
| rule file             | a JSON-formatted rule file placed under $PGDATA that this connector shall apply to its default data type translation rules. See [here](https://docs.synchdb.com/user-guide/transform_rule_file/) for more information    

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
| fields          	| description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      	|
|-----------------	|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------	|
| id              	| unique identifier of a connector slot                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            	|
| connector       	| the type of connector (mysql, oracle, sqlserver...etc)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           	|
| conninfo_name   	| the associated connector info name created by `synchdb_add_conninfo()`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           	|
| pid             	| the PID of the connector worker process                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          	|
| state           	| the state of the connector. Possible states are: <br><br><ul><li>stopped - connector is not running</li><li>initializing - connector is initializing</li><li>paused - connector is paused</li><li>syncing - connector is regularly polling change events</li><li>parsing (the connector is parsing a received change event) </li><li>converting - connector is converting a change event to PostgreSQL representation</li><li>executing - connector is applying the converted change event to PostgreSQL</li><li>updating offset - connector is writing a new offset value to Debezium offset management</li><li>restarting - connector is restarting </li><li>unknown</li></ul> 	|
| err             	| the last error message encountered by the worker which would have caused it to exit. This error could originated from PostgreSQL while processing a change, or originated from Debezium running engine while accessing data from heterogeneous database.                                                                                                                                                                                                                                                                                                                                                                                                                         	|
| last_dbz_offset 	| the last Debezium offset captured by synchdb. Note that this may not reflect the current and real-time offset value of the connector engine. Rather, this is shown as a checkpoint that we could restart from this offeet point if needed.                                                                                                                                                                                                                                                                                                                                                                                                                                       	|
## synchdb_set_offset
Used to set a custom starting offset value:
* `name` - the name of connector to set a new offset
* `offset` - the offset value to set

```
SELECT synchdb_set_offset('mysqlconn', '{"ts_sec":1725644339,"file":"mysql-bin.000004","pos":138466,"row":1,"server_id":223344,"event":2}');
```

## synchdb_restart_connector
Used to restart a connector in a different snapshot mode
* `name` - the name of connector to restart. Must be running already.
* `snapshot_mode` - the snapshot mode to use to restart

This SQL function is useful to quickly restart a connector in different snapshot mode other than `initial` (the default snapshot mode). Refer to the table below for different snapshot modes that we can use:

|  **setting** 	|                                                                                                                             **description**                                                                                                                             	|
|:------------:	|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------:	|
| always       	| The connector performs a snapshot every time that it starts. The snapshot includes the structure and data of the captured tables. After the snapshot completes, the connector begins to stream event records for subsequent database changes.                           	|
| initial      	| The connector performs a database snapshot if not already done. After the snapshot completes, the connector begins to stream event records for subsequent database changes.                                                                                             	|
| initial_only 	| The connector performs a database snapshot. After the snapshot completes, the connector stops, and does not stream event records for subsequent database changes.                                                                                                       	|
| no_data      	| The connector captures the structure of all relevant tables, but not the data they contain.                                                                                                                                                                             	|
| never        	| When the connector starts, rather than performing a snapshot, it immediately begins to stream event records for subsequent database changes.                                                                                                                            	|
| recovery     	| Set this option to restore a database schema history that is lost or corrupted. After a restart, the connector runs a snapshot that rebuilds the topic from the source tables                                                                                           	|
| when_needed  	| After the connector starts, it performs a snapshot only if it detects one of the following circumstances:<br><br><ul><li>It cannot detect any topic offsets</li><li>A previously recorded offset specifies a log position that is not available on the server</li></ul> 	| 

