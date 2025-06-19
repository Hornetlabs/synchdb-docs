# Connector Running State

## Check Connector Running State
Use `synchdb_state_view()` to examine all connectors' running states.

See below for an example output:
``` SQL
postgres=# select * from synchdb_state_view;
     name      | connector_type |  pid   |        stage        |  state  |   err    |                                           last_dbz_offset
---------------+----------------+--------+---------------------+---------+----------+------------------------------------------------------------------------------------------------------
 sqlserverconn | sqlserver      | 579820 | change data capture | polling | no error | {"commit_lsn":"0000006a:00006608:0003","snapshot":true,"snapshot_completed":false}
 mysqlconn     | mysql          | 579845 | change data capture | polling | no error | {"ts_sec":1741301103,"file":"mysql-bin.000009","pos":574318212,"row":1,"server_id":223344,"event":2}
 oracleconn    | oracle         | 580053 | change data capture | polling | no error | offset file not flushed yet
(3 rows)

```

Column Details:

| fields          | description |
|-|-|
| name   | the associated connector info name created by `synchdb_add_conninfo()`|
| connector_type       | the type of connector (mysql, oracle, sqlserver...etc)|
| pid             | the PID of the connector worker process|
| stage           | the stage of the connector worker process|
| state           | the state of the connector. Possible states are: <br><br><ul><li>stopped - connector is not running</li><li>initializing - connector is initializing</li><li>paused - connector is paused</li><li>syncing - connector is regularly polling change events</li><li>parsing (the connector is parsing a received change event) </li><li>converting - connector is converting a change event to PostgreSQL representation</li><li>executing - connector is applying the converted change event to PostgreSQL</li><li>updating offset - connector is writing a new offset value to Debezium offset management</li><li>restarting - connector is restarting </li><li>dumping memory - connector is dumping JVM memory summary in log file </li><li>unknown</li></ul> |
| err             | the last error message encountered by the worker which would have caused it to exit. This error could originated from PostgreSQL while processing a change, or originated from Debezium running engine while accessing data from heterogeneous database. |
| last_dbz_offset | the last Debezium offset captured by synchdb. Note that this may not reflect the current and real-time offset value of the connector engine. Rather, this is shown as a checkpoint that we could restart from this offeet point if needed.|