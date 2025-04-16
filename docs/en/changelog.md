# Change Log
All notable changes to this project will be documented in this file.
 
The format is based on [Keep a Changelog](http://keepachangelog.com/)
and this project adheres to [Semantic Versioning](http://semver.org/).

## **[SynchDB 1.0](https://github.com/Hornetlabs/synchdb/releases/tag/v1.0) - 2024-12-24**
 
This release focuses on bug fixes and performance enhancements following the v1.0 Beta1 release that generally makes it more usable under moderate to high data loads. A lot more Debezium tuning related parameters have also been exposed as PostgreSQL GUCs, allowing user to test with different parameters.
 
### **Added**

* added a data cache in DML parsing stage to prevent frequent access to PostgreSQL's catalog to obtain a table's tuple descriptor structure.
* added a variant of `synchdb_start_engine_bgw(name, mode)` that takes a second argument to indicate a custom snapshot mode to start the connector with. More detail [here](../user-guide/utility_functions).
* added several new GUCs that can be adjusted to tune the performance of Debezium Runner. Refer to [here](../user-guide/configuration.md) for complete list.
* added a debug SQL function `synchdb_log_jvm_meminfo(name)` that causes specified connector to output current JVM heap memory usage summary in PostgreSQL log file.
* added a new VIEW `synchdb_stats_view` that prints statistic information for all connectors. More detail [here](../user-guide/utility_functions).
* added a new SQL function `synchdb_reset_stats(name)` to clear statistic information of specified connector. More detail [here](../user-guide/utility_functions).
* added a mess creation script to quickly generate test tables and data on MySQL database type.

### **Changed**

* synchdb_state_view(): added a new field called `stage` that indicates the current stage of a connector (value can either be `Initial Snapshot` or `Change Data Capture`).
* synchdb_state_view(): will only show state of valid connectors.
* removed sending "partial batch completion" notification to Debezium Runner in case of error, because a batch is now handled by one PostgreSQL transaction, and partial completion is not allowed.
* SSL related parameters per connector can now be specified in the [rule file](../user-guide/transform_rule_file).
* the maximum heap memory to allocate to JVM that runs the Debezium Runner can now be configured via GUC.
* max number of connector background worker is now configurable instead of hardcoded 30.

### **Fixed**

* fixed rapid memory buildups in Debezium runner in JVM by adding a throttle control in the receiving of change events.
* resolved majority of memory leak in both SynchDB and Debezium runner components.
* corrected the use of memory context in SynchDB such that heap memory can be correctly freed at the end of each change event processing.
* significantly increased the processing speed of SynchDB by processing a batch within a single PostgreSQL transaction rather than multiple.
* corrected SQLServer's default data type size mapping for `char` type from 0 to -1.
* resolved high memory usage during DML processing using SPI.

## **[SynchDB 1.0 Beta1](https://github.com/Hornetlabs/synchdb/releases/tag/v1.0_beta1) - 2024-10-23**
 
The first SynchDB beta software release that lays a robust foundation for seamless replication from heterogeneous databases to PostgreSQL.
 
### **Added**

* logical replication from heterogeneous databases: (MySQL and SQLServer).
* [DDL replication](../user-guide/ddl_replication) (CREATE TABLE, DROP TABLE, ALTER TABLE ADD COLUMN, ALTER TABLE DROP COLUMN, ALTER TABLE ALTER COLUMN).
* DML replication (INSERT, UPDATE, DELETE).
* max 30 concurrent connector workers.
* [automatic connector launcher](../user-guide/connector_auto_launcher) at PostgreSQL startup.
* global connector state and last error message views.
* [selective databases and tables replication](../user-guide/selective_table_sync).
* change events in batches.
* connector restarts in different snapshot modes.
* [offset management interfaces](../user-guide/set_offset) to select custom replication resume point.
* default data type and object name transform rules for supported heterogeneous databases.
* [JSON rule file](../user-guide/transform_rule_file) to define custom: (data type, column name, table name and data expression transform rules).
* 2 data apply modes (SPI, HeapAM API).
* several [utility functions](../user-guide/utility_functions) to perform connector operations: (start, stop, pause, resume).
 
### **Changed**

### **Fixed**
