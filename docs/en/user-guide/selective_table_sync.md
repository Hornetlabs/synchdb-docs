# Selective Table Sync

It is possible to select only specific tables from remote heterogeneous database to focus on replication. This could prevent resources being spent on replicating unwanted tables. 

## Select Desired Tables and Start it for the First Time
Table selection is done during connector creation phase via `synchdb_add_conninfo()` where we specify a list of tables (expressed in FQN, separated by a comma) to replicate from.

For example, the following command creates a connector that only replicates change from `inventory.orders` and `inventory.products` tables from remote MySQL database.
```
SELECT synchdb_add_conninfo('mysqlconn','192.168.1.86',3306,'mysqluser', 'mysqlpwd', 'inventory', 'postgres', 'inventory.orders,inventory.products', 'mysql', 'myrule.json');
```

Starting this connector for the very first time will trigger an initial snapshot being performed and selected 2 tables' schema and data will be replicated.

```
SELECT synchdb_start_engine_bgw('mysqlconn');
```

Examine the connector state and the new tables:
```
postgres=# SELECT * FROM synchdb_state_view WHERE conninfo_name='mysqlconn';
 id | connector | conninfo_name |  pid   |  state  |   err    |       last_dbz_offset
----+-----------+---------------+--------+---------+----------+-----------------------------
  0 | mysql     | mysqlconn     | 807536 | syncing | no error | offset file not flushed yet
(1 row)

postgres=# SET search_path TO inventory;
SET
postgres=# \d
                 List of relations
  Schema   |        Name        |   Type   | Owner
-----------+--------------------+----------+--------
 inventory | products           | table    | ubuntu
 inventory | products_id_seq    | sequence | ubuntu
 inventory | orders             | table    | ubuntu
 inventory | orders_ididid_seq  | sequence | ubuntu
 public    | synchdb_conninfo   | table    | ubuntu
 public    | synchdb_state_view | view     | ubuntu
(6 rows)

postgres=#
```

`mysqlconn` connector will then capture subsequent changes applied to `inventory.orders` and `inventory.products` tables.

## Add More Tables to Replicate During Run Time.
The `mysqlconn` from previous section has already completed the initial snapshot and obtained the table schemas of the selected table. If we would like to add more tables to replicate from, we will need to notify the Debezium engine about the updated table section and perform the initial snapshot again. Here's how it is done:

Alter the `synchdb_conninfo` table and update theh table list with more tables to replicate. Here, we add another table `invnetory.customers` to the table sync list:
```
UPDATE synchdb_conninfo SET data = jsonb_set(data, '{table}', '"inventory.orders,inventory.products,inventory.customers"') WHERE name = 'mysqlconn';
```

Restart the connector with a new snapshot mode (always):
```
SELECT synchdb_restart_connector('mysqlconn', 'always');
```

The above command instructs Debezium engine to redo the snapshot, which means that it will re-construct all 3 selected tables even if 2 of them already have the data. 

If the heterogeneous database type does not support DDL replication (such as SQLServer), we may get a data conflict error when the snapshot is being rebuilt on the 2 tables previously selected for replication. If this is the case, we may need to drop or truncate them before restarting the connector with snapshot mode = 'always'.

Now, we can examine our tables again:
```
postgres=# SET search_path TO inventory;
SET
postgres=# \d
                 List of relations
  Schema   |        Name        |   Type   | Owner
-----------+--------------------+----------+--------
 inventory | products           | table    | ubuntu
 inventory | products_id_seq    | sequence | ubuntu
 inventory | orders             | table    | ubuntu
 inventory | orders_ididid_seq  | sequence | ubuntu
 inventory | customers          | table    | ubuntu
 inventory | customers_id_seq   | sequence | ubuntu
 public    | synchdb_conninfo   | table    | ubuntu
 public    | synchdb_state_view | view     | ubuntu
(8 rows)

postgres=#

```

## Snapshot Modes

The table below contains a list of snapshot modes SynchDB can support:

|  **setting** 	|                                                                                                                             **description**                                                                                                                             	|
|:------------:	|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------:	|
| always       	| The connector performs a snapshot every time that it starts. The snapshot includes the structure and data of the captured tables. After the snapshot completes, the connector begins to stream event records for subsequent database changes.                           	|
| initial (default)      	| The connector performs a database snapshot if not already done. After the snapshot completes, the connector begins to stream event records for subsequent database changes.                                                                                             	|
| initial_only 	| The connector performs a database snapshot. After the snapshot completes, the connector stops, and does not stream event records for subsequent database changes.                                                                                                       	|
| no_data      	| The connector captures the structure of all relevant tables, but not the data they contain.                                                                                                                                                                             	|
| never        	| When the connector starts, rather than performing a snapshot, it immediately begins to stream event records for subsequent database changes.                                                                                                                            	|
| recovery     	| Set this option to restore a database schema history that is lost or corrupted. After a restart, the connector runs a snapshot that rebuilds the topic from the source tables                                                                                           	|
| when_needed  	| After the connector starts, it performs a snapshot only if it detects one of the following circumstances:<br><br><ul><li>It cannot detect any topic offsets</li><li>A previously recorded offset specifies a log position that is not available on the server</li></ul> 	| 
