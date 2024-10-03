# Set Custom Start Offset Values

A start offset value represents a point to start replication from in the similar way as PostgreSQL's resume LSN. When Debezium runner engine starts, it will start the replication from this offset value. Setting this offset value to a earlier value will cause Debezium runner engine to start replication from earlier records, possibly replicating duplicate data records. We should be extra cautious when setting start offset values on Debezium.

## Record Settable Offset Values
During operation, new offsets will be generated nd flushed to disk by Debezium runner engine. The last flushed offset can be retrieved from `synchdb_state_view()` utility command:

```
postgres=# select * from synchdb_state_view;
 id | connector | conninfo_name  |  pid   |  state  |   err    |                                          last_dbz_offset
----+-----------+----------------+--------+---------+----------+---------------------------------------------------------------------------------------------------
  0 | mysql     | mysqlconn      | 461696 | syncing | no error | {"ts_sec":1725644339,"file":"mysql-bin.000004","pos":138466,"row":1,"server_id":223344,"event":2}
  1 | sqlserver | sqlserverconn  | 461739 | syncing | no error | {"event_serial_no":1,"commit_lsn":"00000100:00000c00:0003","change_lsn":"00000100:00000c00:0002"}
  3 | null      |                |     -1 | stopped | no error | no offset
  4 | null      |                |     -1 | stopped | no error | no offset
  4 | null      |                |     -1 | stopped | no error | no offset
  5 | null      |                |     -1 | stopped | no error | no offset
  6 | null      |                |     -1 | stopped | no error | no offset
  7 | null      |                |     -1 | stopped | no error | no offset
  8 | null      |                |     -1 | stopped | no error | no offset
  9 | null      |                |     -1 | stopped | no error | no offset
 10 | null      |                |     -1 | stopped | no error | no offset
 11 | null      |                |     -1 | stopped | no error | no offset
 12 | null      |                |     -1 | stopped | no error | no offset
 13 | null      |                |     -1 | stopped | no error | no offset
 14 | null      |                |     -1 | stopped | no error | no offset
 15 | null      |                |     -1 | stopped | no error | no offset
 16 | null      |                |     -1 | stopped | no error | no offset
 17 | null      |                |     -1 | stopped | no error | no offset
 18 | null      |                |     -1 | stopped | no error | no offset
 19 | null      |                |     -1 | stopped | no error | no offset
 20 | null      |                |     -1 | stopped | no error | no offset
 21 | null      |                |     -1 | stopped | no error | no offset
 22 | null      |                |     -1 | stopped | no error | no offset
 23 | null      |                |     -1 | stopped | no error | no offset
 24 | null      |                |     -1 | stopped | no error | no offset
 25 | null      |                |     -1 | stopped | no error | no offset
 26 | null      |                |     -1 | stopped | no error | no offset
 27 | null      |                |     -1 | stopped | no error | no offset
 28 | null      |                |     -1 | stopped | no error | no offset
 29 | null      |                |     -1 | stopped | no error | no offset

```

Depending on the connector type, this offset value differs. From the example above, the `mysql` connector's last flushed offset is `{"ts_sec":1725644339,"file":"mysql-bin.000004","pos":138466,"row":1,"server_id":223344,"event":2}` and `sqlserver`'s last flushed offset is `{"event_serial_no":1,"commit_lsn":"00000100:00000c00:0003","change_lsn":"00000100:00000c00:0002"}`. 

We should save this values regularly, so in case we run into a problem, we know the offset location in the past that can be set to resume the replication operation.


## Pause the Connector
A connector must be in a `paused` state before a new offset value can be set.

Use `synchdb_pause_engine()` SQL function to pause a runnng connector. This will halt the Debezium runner engine from replicating from the heterogeneous database. When paused, it is possible to alter the Debezium connector's offset value to replicate from a specific point in the past using `synchdb_set_offset()` SQL routine. It takes `conninfo_name` as its argument which can be found from the output of `synchdb_get_state()` view.

For example:
```
SELECT synchdb_pause_engine('mysqlconn');
```

## Set the new Offset
Use `synchdb_set_offset()` SQL function to change a connector worker's starting offset. This can only be done when the connector is put into `paused` state. The function takes 2 parameters, `conninfo_name` and `a valid offset string`, both of which can be found from the output of `synchdb_get_state()` view.

For example:
```
SELECT synchdb_set_offset('mysqlconn', '{"ts_sec":1725644339,"file":"mysql-bin.000004","pos":138466,"row":1,"server_id":223344,"event":2}');
```

## Resume the Connector

Use `synchdb_resume_engine()` SQL function to resume Debezium operation from a paused state. This function takes `connector name` as its only parameter, which can be found from the output of `synchdb_get_state()` view. The resumed Debezium runner engine will start the replication from the newly set offset value.

For example:
```
SELECT synchdb_resume_engine('mysqlconn');
```
