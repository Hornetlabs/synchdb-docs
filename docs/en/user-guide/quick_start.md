# Quick Start Guide

It is very simple to start using SynchDB to perform data replication from heterogeneous databases to PostgreSQL given that you have the correct connection information to your heterogeneous databases.

## Install SynchDB Extension
SynchDB extension requires pgcrypto to encrypt certain sensitive credential data. Please make sure it is installed prior to installing SynchDB. Alternatively, you can include `CASCADE` clause in `CREATE EXTENSION` to automatically install dependencies:

```
CREATE EXTENSION synchdb CASCADE;
```

## Create a Connection Info
This can be done with utility SQL function `synchdb_add_conninfo()`.

synchdb_add_conninfo takes these arguments:

* `name` - a unique identifier that represents this connection info
* `hostname` - the IP address of the heterogeneous database.
* `port` - the port number to connect to the heterogeneous database.
* `username` - user name to use.
* `password` - password to authenticate the username.
* `source database` - this is the name of source database in heterogeneous database that we want to replicate changes from.
* `destination` database - this is the name of destination database in PostgreSQL to apply changes to. It must be a valid database that exists in PostgreSQL.
* `table` (optional) - expressed in the form of `[database].[table]` or `[database].[schema].[table]` that must exists in heterogeneous database so the engine will only replicate the specified tables. If left empty, all tables are replicated. Please note that tables not selected here will have their table schema replicated in PostgreSQL as well, however, only the data of the selected table will have their data replicated.
* `connector` - the connector type to use (MySQL, Oracle, SQLServer... etc).
* `rule file` - a JSON-formatted rule file placed under $PGDATA that this connector shall apply to its default data type translation rules.

Examples:

1. Create a MySQL connector called 'mysqlconn' to replicate from source database 'inventory' in MySQL to destination database 'postgres' in PostgreSQL, using rule file `myrule.json`:
```
SELECT synchdb_add_conninfo('mysqlconn', '127.0.0.1', 3306, 'mysqluser', 'mysqlpwd', 'inventory', 'postgres', '', 'mysql', 'myrule.json');
```

2. Create a MySQL connector called 'mysqlconn2' to replicate from source database 'inventory' to destination database 'mysqldb2' in PostgreSQL using default transaltion rule:
```
SELECT synchdb_add_conninfo('mysqlconn2', '127.0.0.1', 3306, 'mysqluser', 'mysqlpwd', 'inventory', 'mysqldb2', '', 'mysql', '');
```

3. Create a SQLServer connector called 'sqlserverconn' to replicate from source database 'testDB' to destination database 'sqlserverdb' in PostgreSQL using default translation rule:
```
SELECT synchdb_add_conninfo('sqlserverconn', '127.0.0.1', 1433, 'sa', 'Password!', 'testDB', 'sqlserverdb', '', 'sqlserver', '');
```

4. Create a MySQL connector called 'mysqlconn3' to replicate from source database 'inventory's `orders` and `customers` tabls to destination database 'mysqldb3' in PostgreSQL using rule file `myrule2.json`:
```
SELECT synchdb_add_conninfo('mysqlconn3', '127.0.0.1', 3306, 'mysqluser', 'mysqlpwd', 'inventory', 'mysqldb3', 'inventory.orders,inventory.customers', 'mysql', 'myrule2.json');
```

## Things to Note
* It is possible to create multiple connectors connecting to the same connector type (ex, MySQL, SQLServer..etc). SynchDB will spawn separate connections to fetch change data.
* Avoid creating 2 connectors connecting to the same heterogeneous database, same source database, tables and the same destination database. This could cause a conflict when both connectors start. In the future, we will add a column and table name mappings feature to allow a source table to be mapped to a different name on the destination to prevent such name conflict. In the meantime, please use different destination database instead.
* User-defined X509 certificate and private key for TLS connection to remote database will be supported in near future. In the meantime, please ensure TLS settings are set to optional.

 

## Check Created Connection Info
All connection information are created in the table `synchdb_conninfo`. We are free to view its content and make modification as required. Please note that the password of a user credential is encrypted by pgcrypto using a key only known to synchdb. So please do not modify the password field or it may be decrypted incorrectly if tempered. See below for an example output:

```
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

```
select synchdb_start_engine_bgw('mysqlconn');
select synchdb_start_engine_bgw('sqlserverconn');
```

## Check Connector Running State
Use `synchdb_state_view()` view to examine all the running connectors and their states. Currently, synchdb can support up to 30 running workers.

See below for an example output:
```
postgres=# select * from synchdb_state_view;
 id | connector | conninfo_name  |  pid   |  state  |   err    |                                          last_dbz_offset
----+-----------+----------------+--------+---------+----------+---------------------------------------------------------------------------------------------------
  0 | mysql     | mysqlconn      | 461696 | syncing | no error | {"ts_sec":1725644339,"file":"mysql-bin.000004","pos":138466,"row":1,"server_id":223344,"event":2}
  1 | sqlserver | sqlserverconn  | 461739 | syncing | no error | {"event_serial_no":1,"commit_lsn":"00000100:00000c00:0003","change_lsn":"00000100:00000c00:0002"}
  3 | null      |                |     -1 | stopped | no error | no offset
  4 | null      |                |     -1 | stopped | no error | no offset
  4 | null      |                |     -1 | stopped | no error | no offset
  5 | null      |                |     -1 | stopped | no error | no offset
  6 | null      |                |     -1 | stopped | no error | no offset
  7 | null      |                |     -1 | stopped | no error | no offset
  8 | null      |                |     -1 | stopped | no error | no offset
  9 | null      |                |     -1 | stopped | no error | no offset
 10 | null      |                |     -1 | stopped | no error | no offset
 11 | null      |                |     -1 | stopped | no error | no offset
 12 | null      |                |     -1 | stopped | no error | no offset
 13 | null      |                |     -1 | stopped | no error | no offset
 14 | null      |                |     -1 | stopped | no error | no offset
 15 | null      |                |     -1 | stopped | no error | no offset
 16 | null      |                |     -1 | stopped | no error | no offset
 17 | null      |                |     -1 | stopped | no error | no offset
 18 | null      |                |     -1 | stopped | no error | no offset
 19 | null      |                |     -1 | stopped | no error | no offset
 20 | null      |                |     -1 | stopped | no error | no offset
 21 | null      |                |     -1 | stopped | no error | no offset
 22 | null      |                |     -1 | stopped | no error | no offset
 23 | null      |                |     -1 | stopped | no error | no offset
 24 | null      |                |     -1 | stopped | no error | no offset
 25 | null      |                |     -1 | stopped | no error | no offset
 26 | null      |                |     -1 | stopped | no error | no offset
 27 | null      |                |     -1 | stopped | no error | no offset
 28 | null      |                |     -1 | stopped | no error | no offset
 29 | null      |                |     -1 | stopped | no error | no offset
```

Column Details:
* id: unique identifier of a connector slot
* connector: the type of connector (mysql, oracle, sqlserver...etc)
* conninfo_name: the associated connection info name created by `synchdb_add_conninfo()`
* pid: the PID of the connector worker process
* state: the state of the connector. Possible states are:
    * stopped
    * initializing
    * paused
    * syncing
    * parsing
    * converting
    * executing
    * updating offset
    * unknown
* err: the last error message encountered by the worker which would have caused it to exit. This error could originated from PostgreSQL while processing a change, or originated from Debezium running engine while accessing data from heterogeneous database.
* last_dbz_offset: the last Debezium offset captured by synchdb. Note that this may not reflect the current and real-time offset value of the connector engine. Rather, this is shown as a checkpoint that we could restart from this offeet point if needed.

## Stop a Connector
Use `synchdb_stop_engine_bgw()` SQL function to stop a running or paused connector worker. This function takes `conninfo_name` as its only parameter, which can be found from the output of `synchdb_get_state()` view.

For example:
```
select synchdb_stop_engine_bgw('mysqlconn');
```

`synchdb_stop_engine_bgw()` function also marks a connection info as `inactive`, which prevents the this worker from automatic-relaunch at server restarts. See below for more details.

