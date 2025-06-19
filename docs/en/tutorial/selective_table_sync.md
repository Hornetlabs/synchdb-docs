---
weight: 90
---
# Selective Table Synchronization

It is possible to select only specific tables from remote heterogeneous database to focus on replication. This could prevent resources being spent on replicating unwanted tables. 

## **Select Desired Tables and Start it for the First Time**

Table selection is done during connector creation phase via `synchdb_add_conninfo()` where we specify a list of tables (expressed in FQN, separated by a comma) to replicate from.

For example, the following command creates a connector that only replicates change from `inventory.orders` and `inventory.products` tables from remote MySQL database.
```sql
SELECT synchdb_add_conninfo(
    'mysqlconn', 
    '192.168.1.86', 
    3306, 
    'mysqluser', 
    'mysqlpwd', 
    'inventory', 
    'postgres', 
    'inventory.orders,inventory.products', 
    'mysql'
);
```

Starting this connector for the very first time will trigger an initial snapshot being performed and selected 2 tables' schema and data will be replicated.

```sql
SELECT synchdb_start_engine_bgw('mysqlconn');
```

### **Verify the Connector State and Tables**

Examine the connector state and the new tables:
```sql
postgres=# Select name, state, err from synchdb_state_view;
     name      |  state  |   err
---------------+---------+----------
 mysqlconn     | polling | no error
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

Once the snapshot is complete, the `mysqlconn` connector will continue capturing subsequent changes to the `inventory.orders` and `inventory.products` tables.

## **Add More Tables to Replicate During Run Time.**

The `mysqlconn` from previous section has already completed the initial snapshot and obtained the table schemas of the selected table. If we would like to add more tables to replicate from, we will need to notify the Debezium engine about the updated table section and perform the initial snapshot again. Here's how it is done:

1. Update the `synchdb_conninfo` table to include additional tables.
2. In this example, we add the `inventory.customers` table to the sync list:
```sql
UPDATE synchdb_conninfo 
SET data = jsonb_set(data, '{table}', '"inventory.orders,inventory.products,inventory.customers"') 
WHERE name = 'mysqlconn';
```
3. Configure the snapshot table parameter to include only the new table `inventory.customers` to that SynchDB does not try to rebuild the 2 tables that have already finished the snapshot.
```sql
UPDATE synchdb_conninfo 
SET data = jsonb_set(data, '{snapshottable}', '"inventory.customers"') 
WHERE name = 'mysqlconn';
``` 
4. Restart the connector with the snapshot mode set to `always` to perform another initial snapshot:
```sql
SELECT synchdb_restart_connector('mysqlconn', 'always');
```
This forces Debezium to re-snapshot only the new table `inventory.customers` while leaving the old tables `inventory.orders` and `inventory.products` untouched. The CDC for all tables will resume once snapshot is complete. 


### **Verify the Updated Tables**

Now, we can examine our tables again:
```sql
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

## **Snapshot Modes**

SynchDB offers different snapshot modes, depending on your replication needs:

|  **setting** |**description**|
|:-:|-|
| always       | The connector performs a snapshot every time that it starts. The snapshot includes the structure and data of the captured tables. After the snapshot completes, the connector begins to stream event records for subsequent database changes.|
| initial (default)      | The connector performs a database snapshot if not already done. After the snapshot completes, the connector begins to stream event records for subsequent database changes.|
| initial_only | The connector performs a database snapshot. After the snapshot completes, the connector stops, and does not stream event records for subsequent database changes.|
| no_data      | The connector captures the structure of all relevant tables, but not the data they contain.|
| never        | When the connector starts, rather than performing a snapshot, it immediately begins to stream event records for subsequent database changes.|
| recovery     | Set this option to restore a database schema history that is lost or corrupted. After a restart, the connector runs a snapshot that rebuilds the topic from the source tables|
| when_needed  | After the connector starts, it performs a snapshot only if it detects one of the following circumstances:<br><ul><li>It cannot detect any topic offsets</li><li>A previously recorded offset specifies a log position that is not available on the server</li></ul> | 
