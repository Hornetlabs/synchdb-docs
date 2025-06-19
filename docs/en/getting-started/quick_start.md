---
weight: 30
---
# Quick Start Guide

It is very simple to start using SynchDB to perform data replication from heterogeneous databases to PostgreSQL given that you have the correct connection information to your heterogeneous databases. Make sure you have already done the installation steps described [here](https://docs.synchdb.com/user-guide/installation/) and prepared [sample databases](https://docs.synchdb.com/user-guide/prepare_tests_env/) to test if you have not already.

## **Create SynchDB Extension**

SynchDB extension requires pgcrypto to encrypt certain sensitive credential data. Please make sure it is installed prior to installing SynchDB. Alternatively, you can include `CASCADE` clause in `CREATE EXTENSION` to automatically install dependencies:

```sql
CREATE EXTENSION synchdb CASCADE;
```

## **Create a Connector**

Here are some examples to create a basic connector for each supported source database type.

1. Create a MySQL connector called `mysqlconn` to replicate all tables under `inventory` in MySQL to destination database `postgres` in PostgreSQL:
```sql
SELECT synchdb_add_conninfo(
    'mysqlconn', '127.0.0.1', 3306, 'mysqluser', 
    'mysqlpwd', 'inventory', 'postgres', 
    'null', 'null', 'mysql');
```

2. Create a SQLServer connector called `sqlserverconn` to replicate all tables under `testDB` to destination database 'postgres' in PostgreSQL:
```sql
SELECT 
  synchdb_add_conninfo(
    'sqlserverconn', '127.0.0.1', 1433, 
    'sa', 'Password!', 'testDB', 'postgres', 
    'null', 'null', 'sqlserver');
```

3. Create a Oracle connector called `oracleconn` to replicate all tables under `FREE` to destination database 'postgres' in PostgreSQL:
```sql
SELECT 
  synchdb_add_conninfo(
    'oracleconn', '127.0.0.1', 1521, 
    'c##dbzuser', 'dbz', 'FREE', 'postgres', 
    'null', 'null', 'oracle');
```

Successful creation will have a record under `synchdb_conninfo` table. More details on creating a connector can be found [here](https://docs.synchdb.com/user-guide/create_a_connector/)

## **Start a Connector**

Use `synchdb_start_engine_bgw()` function to start a connector worker. It takes one argument which is the connection  name created above. This command will spawn a new background worker to connect to the heterogeneous database with the specified configurations.

For example, the following will spawn 2 background worker in PostgreSQL, one replicating from a MySQL database, the other from SQL Server

```sql
select synchdb_start_engine_bgw('mysqlconn');
select synchdb_start_engine_bgw('sqlserverconn');
select synchdb_start_engine_bgw('oracleconn');
```

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

## **Check Connector Running Statistics**

Use `synchdb_stats_view()` view to examine the statistic information of all connectors. These statistics record cumulative measurements about different types of change events a connector has processed so far. Currently these statistic values are stored in shared memory and not persisted to disk. Persist statistics data is a feature to be added in near future. The statistic information is updated at every successful completion of a batch and it contains several timestamps of the first and last change event within that batch. By looking at these timestamps, we can roughly tell the time it takes to finish processing the batch and the delay between when the data is generated, and processed by both Debezium and PostgreQSL.

See below for an example output:
```sql
postgres=# select * from synchdb_stats_view;
    name    | ddls | dmls | reads | creates | updates | deletes | bad_events | total_events | batches_done | avg_batch_size | first_src_ts  | first_dbz_ts  |  first_pg_ts  |  last_src_ts  |  last_dbz_ts  |  last_pg_ts
------------+------+------+-------+---------+---------+---------+------------+--------------+--------------+----------------+---------------+---------------+---------------+---------------+---------------+---------------
 oracleconn |    1 |    1 |     0 |       1 |       0 |       0 |          0 |            2 |            1 |              2 | 1744398189000 | 1744398230893 | 1744398231243 | 1744398198000 | 1744398230950 | 1744398231244
(1 row)
```

Column Details:

| fields | description |
|-|-|
| name | the associated connector info name created by `synchdb_add_conninfo()`|
| ddls | number of DDLs operations completed |
| dmls | number of DMLs operations completed |
| reads | number of READ events completed during initial snapshot stage |
| creates | number of CREATES events completed during CDC stage |
| updates | number of UPDATES events completed during CDC stage |
| deletes | number of DELETES events completed during CDC stage |
| bad_events | number of bad events ignored (such as empty events, unsupported DDL events..etc) |
| total_events | total number of events processed (including bad_events) |
| batches_done | number of batches completed |
| avg_batch_size | average batch size (total_events / batches_done) |
| first_src_ts | the timestamp in nanoseconds when the last batch's first event is produced at the external database |
| first_dbz_ts | the timestamp in nanoseconds when the last batch's first event is processed by Debezium Engine |
| first_pg_ts | the timestamp in nanoseconds when the last batch's first event is applied to PostgreSQL |
| last_src_ts | the timestamp in nanoseconds when the last batch's last event is produced at the external database |
| last_dbz_ts | the timestamp in nanoseconds when the last batch's last event is processed by Debezium Engine |
| last_pg_ts | the timestamp in nanoseconds when the last batch's last event is applied to PostgreSQL |


## **Stop a Connector**

Use `synchdb_stop_engine_bgw()` SQL function to stop a running or paused connector worker. This function takes `conninfo_name` as its only parameter, which can be found from the output of `synchdb_get_state()` view.

For example:
```
select synchdb_stop_engine_bgw('mysqlconn');
```

`synchdb_stop_engine_bgw()` function also marks a connection info as `inactive`, which prevents the this worker from automatic-relaunch at server restarts. See below for more details.

## **Remove a Connector**
Use `synchdb_del_conninfo()` SQL function to remove a connector from SynchDB. This will wipe out the [metadata files](https://docs.synchdb.com/architecture/metadata_files/) and also any [object mappings](https://docs.synchdb.com/user-guide/object_mapping_rules/) created on that connector.

For example:
```
select synchdb_del_conninfo('mysqlconn');
```