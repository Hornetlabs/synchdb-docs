# Connector Statistics

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


**Note: the statistic information is not persistent and will be wiped if PostgreSQL shuts down or restarts**

## **synchdb_reset_stats**

**Purpose**: Resets all statistic information of given connector name

```sql
SELECT synchdb_reset_stats('mysqlconn');
```
