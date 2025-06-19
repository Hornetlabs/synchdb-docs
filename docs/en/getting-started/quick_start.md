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

Use `synchdb_start_engine_bgw()` function to start a connector worker. It takes one argument which is the connection name created above. This command will start the connector in the default snapshot mode called `initial`, which will replicate the schema of all designated tables and their initial data in PostgreSQL.

```sql
select synchdb_start_engine_bgw('mysqlconn');
select synchdb_start_engine_bgw('sqlserverconn');
select synchdb_start_engine_bgw('oracleconn');
```

## Check Connector Running State
Use `synchdb_state_view()` to examine all connectors' running states. 

If everything works fine, we should see outputs similar to below.
``` SQL
postgres=# select * from synchdb_state_view;
     name      | connector_type |  pid   |        stage        |  state  |   err    |                                           last_dbz_offset
---------------+----------------+--------+---------------------+---------+----------+------------------------------------------------------------------------------------------------------
 sqlserverconn | sqlserver      | 579820 | change data capture | polling | no error | {"commit_lsn":"0000006a:00006608:0003","snapshot":true,"snapshot_completed":false}
 mysqlconn     | mysql          | 579845 | change data capture | polling | no error | {"ts_sec":1741301103,"file":"mysql-bin.000009","pos":574318212,"row":1,"server_id":223344,"event":2}
 oracleconn    | oracle         | 580053 | change data capture | polling | no error | offset file not flushed yet
(3 rows)

```

If there is an error during connector start, perhaps due to connectivity, incorrect user credentials or source database settings, the error message shall be captured and displayed as `err` in the connector state output.

More on running states [here](https://docs.synchdb.com/monitoring/state_view/), and also running statistics [here](https://docs.synchdb.com/monitoring/stats_view/).

## Check the Tables and Data
By default, the source tables will be replicated to a new schema under the current postgreSQL database with the same name as the source database name. We can update the search path to view the new tables in different schema.

For example:
```sql
postgres=# set search_path=public,free,inventory;
SET
postgres=# \d
                 List of relations
  Schema   |        Name        |   Type   | Owner
-----------+--------------------+----------+--------
 free      | orders             | table    | ubuntu
 inventory | addresses          | table    | ubuntu
 inventory | addresses_id_seq   | sequence | ubuntu
 inventory | customers          | table    | ubuntu
 inventory | customers_id_seq   | sequence | ubuntu
 inventory | geom               | table    | ubuntu
 inventory | geom_id_seq        | sequence | ubuntu
 inventory | perf_test_1        | table    | ubuntu
 inventory | products           | table    | ubuntu
 inventory | products_id_seq    | sequence | ubuntu
 inventory | products_on_hand   | table    | ubuntu
 public    | synchdb_att_view   | view     | ubuntu
 public    | synchdb_attribute  | table    | ubuntu
 public    | synchdb_conninfo   | table    | ubuntu
 public    | synchdb_objmap     | table    | ubuntu
 public    | synchdb_state_view | view     | ubuntu
 public    | synchdb_stats_view | view     | ubuntu
(17 rows)
```

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