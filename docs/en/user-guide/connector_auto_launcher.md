# Connector Auto Launcher

## Enable SynchDB Auto Launcher
A connection worker becomes eligible for automatic Launch when `synchdb_start_engine_bgw()` is issued on a particular `connector name`. Likewise, it becomes ineligible when `synchdb_stop_engine_bgw()` is issued. 

Automatic connector launcher can be enabled by:

* add `synchdb` to `shared_preload_libraries` GUC option in postgresql.conf
* set new GUC option `synchdb.synchdb_auto_launcher` to true in postgresql.conf
* restart the PostgreSQL server for the changes to take effect

For example:
```
shared_preload_libraries = 'synchdb'
synchdb.synchdb_auto_launcher = true
```

At startup, SynchDB extension will be preloaded very early. With `synchdb.synchdb_auto_launcher` set to true, SynchDB will spawn a `synchdb_auto_launcher` background worker that will retrieve all the conninfos in `synchdb_conninfo` table that is marked as `active` ( has `isactive` flag set to `true`). Then, it will start them automatically as a separete background worker in the same way as when `synchdb_start_engine_bgw()` is called.. `synchdb_auto_launcher` will exit after.

## Known Issue
`synchdb_auto_launcher` worker will login to the default `postgres` database and try to find active conenctors from `synchdb_conninfo` table. If SynchDB has been installed from non-default database, then `synchdb_auto_launcher` will fail to find the table, and thus not automatically starting the connector worker. In the future, we will make `synchdb_auto_launcher` check for all the databases and automatically start conenctor workers based on every database's `synchdb_conninfo` tables.
