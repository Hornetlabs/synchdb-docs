---
weight: 100
---
# Custom Start Offset Values

A start offset value represents a point to start replication from in the similar way as PostgreSQL's resume LSN. When Debezium runner engine starts, it will start the replication from this offset value. Setting this offset value to a earlier value will cause Debezium runner engine to start replication from earlier records, possibly replicating duplicate data records. We should be extra cautious when setting start offset values on Debezium.

## **Record Settable Offset Values**

During operation, new offsets will be generated nd flushed to disk by Debezium runner engine. The last flushed offset can be retrieved from `synchdb_state_view()` utility command:

```sql
postgres=# select name, last_dbz_offset from synchdb_state_view;
     name      |                                           last_dbz_offset
---------------+------------------------------------------------------------------------------------------------------
 sqlserverconn | {"commit_lsn":"0000006a:00006608:0003","snapshot":true,"snapshot_completed":false}
 mysqlconn     | {"ts_sec":1741301103,"file":"mysql-bin.000009","pos":574318212,"row":1,"server_id":223344,"event":2}
 oracleconn    | offset file not flushed yet
(3 rows)

```

Depending on the connector type, this offset value differs. From the example above, the `mysql` connector's last flushed offset is `{"ts_sec":1741301103,"file":"mysql-bin.000009","pos":574318212,"row":1,"server_id":223344,"event":2}` and `sqlserver`'s last flushed offset is `{"commit_lsn":"0000006a:00006608:0003","snapshot":true,"snapshot_completed":false}`. 

We should save this values regularly, so in case we run into a problem, we know the offset location in the past that can be set to resume the replication operation.


## **Pause the Connector**

A connector must be in a `paused` state before a new offset value can be set.

Use `synchdb_pause_engine()` SQL function to pause a runnng connector. This will halt the Debezium runner engine from replicating from the heterogeneous database. When paused, it is possible to alter the Debezium connector's offset value to replicate from a specific point in the past using `synchdb_set_offset()` SQL routine. It takes `conninfo_name` as its argument which can be found from the output of `synchdb_get_state()` view.

For example:
```sql
SELECT synchdb_pause_engine('mysqlconn');
```

## **Set the new Offset**

Use `synchdb_set_offset()` SQL function to change a connector worker's starting offset. This can only be done when the connector is put into `paused` state. The function takes 2 parameters, `conninfo_name` and `a valid offset string`, both of which can be found from the output of `synchdb_get_state()` view.

For example:
```sql
SELECT 
  synchdb_set_offset(
    'mysqlconn', '{"ts_sec":1741301103,"file":"mysql-bin.000009","pos":574318212,"row":1,"server_id":223344,"event":2}'
  );
```

## **Resume the Connector**

Use `synchdb_resume_engine()` SQL function to resume Debezium operation from a paused state. This function takes `connector name` as its only parameter, which can be found from the output of `synchdb_get_state()` view. The resumed Debezium runner engine will start the replication from the newly set offset value.

For example:
```sql
SELECT synchdb_resume_engine('mysqlconn');
```
