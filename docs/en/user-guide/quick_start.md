---
weight: 30
---
# Quick Start Guide

It is very simple to start using SynchDB to perform data replication from heterogeneous databases to PostgreSQL given that you have the correct connection information to your heterogeneous databases.

## Install SynchDB Extension
SynchDB extension requires pgcrypto to encrypt certain sensitive credential data. Please make sure it is installed prior to installing SynchDB. Alternatively, you can include `CASCADE` clause in `CREATE EXTENSION` to automatically install dependencies:

```sql
CREATE EXTENSION synchdb CASCADE;
```

## Create a Connection Info
This can be done with utility SQL function `synchdb_add_conninfo()`.

synchdb_add_conninfo takes these arguments:

|        argumet        | description |
|-------------------- |-|
| name                  | a unique identifier that represents this connector info |
| hostname              | the IP address or hostname of the heterogeneous database. |
| port                  | the port number to connect to the heterogeneous database. |
| username              | user name to use to authenticate with heterogeneous database.|
| password              | password to authenticate the username |
| source database       | this is the name of source database in heterogeneous database that we want to replicate changes from.|
| destination database  | this is the name of destination database in PostgreSQL to apply changes to. It must be a valid database that exists in PostgreSQL.|
| table                 | (optional) - expressed in the form of `[database].[table]` or `[database].[schema].[table]` that must exists in heterogeneous database so the engine will only replicate the specified tables. If left empty, all tables are replicated.  |
| connector             | the connector type to use (MySQL, Oracle, SQLServer... etc).|
| rule file             | a JSON-formatted rule file placed under $PGDATA that this connector shall apply to its default data type translation rules. See [here](https://docs.synchdb.com/user-guide/transform_rule_file/) for more information.|

Examples:

1. Create a MySQL connector called `mysqlconn` to replicate from source database `inventory` in MySQL to destination database `postgres` in PostgreSQL, using rule file `myrule.json`:
```sql
SELECT synchdb_add_conninfo(
    'mysqlconn',
    '127.0.0.1',
    3306,
    'mysqluser',
    'mysqlpwd',
    'inventory',
    'postgres',
    '',
    'mysql',
    'myrule.json');
```

2. Create a MySQL connector called `mysqlconn2` to replicate from source database `inventory` to destination database `mysqldb2` in PostgreSQL using default transaltion rule:
```sql
SELECT synchdb_add_conninfo(
    'mysqlconn2', '127.0.0.1', 3306, 'mysqluser', 
    'mysqlpwd', 'inventory', 'mysqldb2', 
    '', 'mysql', ''
  );
```

3. Create a SQLServer connector called 'sqlserverconn' to replicate from source database 'testDB' to destination database 'sqlserverdb' in PostgreSQL using default translation rule:
```sql
SELECT 
  synchdb_add_conninfo(
    'sqlserverconn', '127.0.0.1', 1433, 
    'sa', 'Password!', 'testDB', 'sqlserverdb', 
    '', 'sqlserver', ''
  );
```

4. Create a MySQL connector called `mysqlconn3` to replicate from source database `inventory`s `orders` and `customers` tabls to destination database `mysqldb3` in PostgreSQL using rule file `myrule2.json`:
```sql
SELECT 
  synchdb_add_conninfo(
    'mysqlconn3', '127.0.0.1', 3306, 'mysqluser', 
    'mysqlpwd', 'inventory', 'mysqldb3', 
    'inventory.orders,inventory.customers', 
    'mysql', 'myrule2.json'
  );
```

## Things to Note
* It is possible to create multiple connectors connecting to the same connector type (ex, MySQL, SQLServer..etc). SynchDB will spawn separate connections to fetch change data.
* User-defined X509 certificate and private key for TLS connection to remote database will be supported in near future. In the meantime, please ensure TLS settings are set to optional.
* If SSL is required to establish the connection, refer to [here](https://docs.synchdb.com/user-guide/transform_rule_file/) to learn how to configure SSL settings in the rule file per conenctor
 

## Check Created Connection Info
All connection information are created in the table `synchdb_conninfo`. We are free to view its content and make modification as required. Please note that the password of a user credential is encrypted by pgcrypto using a key only known to synchdb. So please do not modify the password field or it may be decrypted incorrectly if tempered. See below for an example output:

```sql
postgres=# \x
Expanded display is on.

postgres=# select * from synchdb_conninfo;
-[ RECORD 1 ]-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
name | mysqlconn
data | {"pwd": "\\xc30d040703024828cc4d982e47b07bd23901d03e40da5995d2a631fb89d49f748b87247aee94070f71ecacc4990c3e71cad9f68d57c440de42e35bcc78fd145feab03452e454284289db", "port": 3306, "user": "mysqluser", "dstdb": "postgres", "srcdb": "inventory", "table": "null", "hostname": "192.168.1.86", "connector": "mysql"i, "myrule.json"}
-[ RECORD 2 ]-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
name | sqlserverconn
data | {"pwd": "\\xc30d0407030231678e1bb0f8d3156ad23a010ca3a4b0ad35ed148f8181224885464cdcfcec42de9834878e2311b343cd184fde65e0051f75d6a12d5c91d0a0403549fe00e4219215eafe1b", "port": 1433, "user": "sa", "dstdb": "sqlserverdb", "srcdb": "testDB", "table": "null", "hostname": "192.168.1.86", "connector": "sqlserver", "null"}
```

## Start a Connector
Use `synchdb_start_engine_bgw()` function to start a connector worker. It takes one argument which is the connection  name created above. This command will spawn a new background worker to connect to the heterogeneous database with the specified configurations.

For example, the following will spawn 2 background worker in PostgreSQL, one replicating from a MySQL database, the other from SQL Server

```sql
select synchdb_start_engine_bgw('mysqlconn');
select synchdb_start_engine_bgw('sqlserverconn');
```

## Check Connector Running State
Use `synchdb_state_view()` view to examine all the running connectors and their states. Currently, synchdb can support up to 30 running workers.

See below for an example output:
```sql
postgres=# select * from synchdb_state_view;
 id | connector | conninfo_name  |  pid   |  state  |   err    |                                          last_dbz_offset
----+-----------+----------------+--------+---------+----------+---------------------------------------------------------------------------------------------------
  0 | mysql     | mysqlconn      | 461696 | syncing | no error | {"ts_sec":1725644339,"file":"mysql-bin.000004","pos":138466,"row":1,"server_id":223344,"event":2}
  1 | sqlserver | sqlserverconn  | 461739 | syncing | no error | {"event_serial_no":1,"commit_lsn":"00000100:00000c00:0003","change_lsn":"00000100:00000c00:0002"}
  2 | null      |                |     -1 | stopped | no error | no offset
  3 | null      |                |     -1 | stopped | no error | no offset
  4 | null      |                |     -1 | stopped | no error | no offset
  5 | null      |                |     -1 | stopped | no error | no offset
  ...
  ...
```


Column Details:

| fields          | description |
|-|-|
| id              | unique identifier of a connector slot|
| connector       | the type of connector (mysql, oracle, sqlserver...etc)|
| name   | the associated connector info name created by `synchdb_add_conninfo()`|
| pid             | the PID of the connector worker process|
| state           | the state of the connector. Possible states are: <br><br><ul><li>stopped - connector is not running</li><li>initializing - connector is initializing</li><li>paused - connector is paused</li><li>syncing - connector is regularly polling change events</li><li>parsing (the connector is parsing a received change event) </li><li>converting - connector is converting a change event to PostgreSQL representation</li><li>executing - connector is applying the converted change event to PostgreSQL</li><li>updating offset - connector is writing a new offset value to Debezium offset management</li><li>restarting - connector is restarting </li><li>dumping memory - connector is dumping JVM memory summary in log file </li><li>unknown</li></ul> |
| err             | the last error message encountered by the worker which would have caused it to exit. This error could originated from PostgreSQL while processing a change, or originated from Debezium running engine while accessing data from heterogeneous database. |
| last_dbz_offset | the last Debezium offset captured by synchdb. Note that this may not reflect the current and real-time offset value of the connector engine. Rather, this is shown as a checkpoint that we could restart from this offeet point if needed.|

## Check Connector Running Statistics
Use `synchdb_stats_view()` view to examine the statistic information of all connectors. These statistics record cumulative measurements about different types of change events a connector has processed so far. Currently these statistic values are stored in shared memory and not persisted to disk. Persist statistics data is a feature to be added in near future.

See below for an example output:
```sql
postgres=# select * from synchdb_stats_view;
   connector   | ddls |  dmls   |  reads  | creates | updates | deletes | bad_events | total_events | batches_done | avg_batch_size
---------------+------+---------+---------+---------+---------+---------+------------+--------------+--------------+----------------
 mysqltpccconn |   22 | 3887111 | 3263746 |  208684 |  400241 |   14440 |      14444 |      3901573 |         2441 |           1598
               |    0 |       0 |       0 |       0 |       0 |       0 |          0 |            0 |            0 |              0
               |    0 |       0 |       0 |       0 |       0 |       0 |          0 |            0 |            0 |              0
               |    0 |       0 |       0 |       0 |       0 |       0 |          0 |            0 |            0 |              0
               |    0 |       0 |       0 |       0 |       0 |       0 |          0 |            0 |            0 |              0
  ...
  ...
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


## Stop a Connector
Use `synchdb_stop_engine_bgw()` SQL function to stop a running or paused connector worker. This function takes `conninfo_name` as its only parameter, which can be found from the output of `synchdb_get_state()` view.

For example:
```
select synchdb_stop_engine_bgw('mysqlconn');
```

`synchdb_stop_engine_bgw()` function also marks a connection info as `inactive`, which prevents the this worker from automatic-relaunch at server restarts. See below for more details.

