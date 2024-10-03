# Metadata Files
During Operation, Debezium Runner engine produces metadata files under $PGDATA/pg_synchdb. Currently 2 types of metadata files are generated and persisted:
* offset file: contains the offset to resume the replication operation at start up
* schema history file: contains the schema information to build all the tables for a replication. This is created during the initial data snapshot sync and can be updated during operation.

These metadata filenames consist of:
* (connector type)_(connector name)_offsets.dat
* (connector type)_(connector name)_schemahistory.dat

```
ls $PGDATA/pg_synchdb
mysql_mysqlconn_offsets.dat        sqlserver_sqlserverconn_offsets.dat
mysql_mysqlconn_schemahistory.dat  sqlserver_sqlserverconn_schemahistory.dat
```

These binary files' contents can be viewed with hexdump command:
```
hexdump -C $PGDATA/pg_synchdb/mysql_mysqlconn_offsets.dat
hexdump -C $PGDATA/pg_synchdb/mysql_mysqlconn_schemahistory.dat
```
