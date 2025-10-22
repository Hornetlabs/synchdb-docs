# Configure Snapshot Engine

## **Initial Snapshot vs Change Data Capture (CDC)**

Initial snapshot refers to the process of migrating both the table schema and the initial data from remote database to SynchDB. For most connector types, this is done only once on first connector start via embedded Debezium runner as the only engine that can perform initial snapshot. After the initial snapshot completes, the Change Data Capture will begin to stream live changes to SynchDB. The "snapshot modes" can further control the behavior of initial snapshot and CDC. Refer to [here](https://docs.synchdb.com/user-guide/start_stop_connector/) for more information.

In addition to Debezium based initial snapshot, which may be slow when there is a huge number of tables, SynchdB provides an alternative FDW-based native initial snapshot engine. FDW based snapshot is only supported in native Openlog Replicator (OLR) connector as of now. All other connectors still rely on debezium to carry out the snapshot.


## **FDW Based Initial Snapshot**

* Enabled in `postgresql.conf` by setting "synchdb.olr_snapshot_engine" to "fdw"
* Start a OLR connector with a snapshot mode that requires doing a snapshot. For example:

```sql
SELECT synchdb_start_engine_bgw('olrconn', 'no_data');

-- or
SELECT synchdb_start_engine_bgw('olrconn', 'always');

-- or
SELECT synchdb_start_engine_bgw('olrconn', 'schemasync');

```

## **Build and Install oracle_fdw**





