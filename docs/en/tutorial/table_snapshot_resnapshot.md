# Table Snapshot and Re-snapshot

## Initial Snapshot
"Initial snapshot" (or table snapshot) in SynchDB means to copy table schema plus initial data for all designated tables. This is similar to the term "table sync" in PostgreSQL logical replication. When a connector is started using the default `initial` mode, it will automatically perform the initial snapshot before going to Change Data Capture (CDC) stage. This can be omitted entirely with mode `never` or partially omitted with mode `no_data`. See [here](../../user-guide/start_stop_connector/) for all snapshot options.

Once the initial snapshot is completed, the connector will not do it again upon subsequent restarts and will just resume with CDC since the last incomplete offset. This behavior is controled by the metadata files managed by Debezium engine. See [here](../../architecture/metadata_files/) for more about metadata files.


## Re-snapshot
If for any reason, user needs to perform the initial snapshot again to re-build `all the designated tables` and all of the initial data, we need to use the `always` snapshot mode, which causes the connector to obtain schema and initial data again at the moment the connector starts. You may need to drop all the desginated tables or clears ll the data in them before SynchDB will attempt to create the tables and populate initial data again, which may exist already. 

Please be cautious as this may be an aggressive action to perform in your setups. A better alternative would be a selective snapshot where only the selected tables will be re-snapshotted in `always` snapshot mode. See below.


## Selective Snapshot
Selective snapshot can be configured to a connector during creation of changed in run time. This done by specifying a list of tables to perform snapshot in `snapshot table` paramter. For example:


**During creation:**
This example creates a conenctor that perform CDC on `inventory.orders`,`inventory.customers` and `invnetory.produts` tables but will only do initial snapshot again for `inventory.products` if the connector starts in `always` snapshot mode.

```sql
SELECT synchdb_add_conninfo(
    'mysqlconn',
    '127.0.0.1',
    3306,
    'mysqluser', 
    'mysqlpwd',
    'inventory',
    'postgres', 
    'inventory.orders,inventory.customers,invnetory.produts',
    'inventory.products',
    'mysql');

SELECT synchdb_start_engine_bgw('mysqlconn', 'always');
```

**Alter existing connector:**
This example sets `inventory.products` to snapshot table field. When started in `always` mode, only `inventory.products` table will be re-snapshotted.

```sql
UPDATE synchdb_conninfo
	SET data = jsonb_set(data, '{snapshottable}', 'inventory.products', true) 
	WHERE name = 'mysqlconn';

SELECT synchdb_start_engine_bgw('mysqlconn', 'always');
```