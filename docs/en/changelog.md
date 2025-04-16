# Change Log
All notable changes to this project will be documented in this file.
 
The format is based on [Keep a Changelog](http://keepachangelog.com/)
and this project adheres to [Semantic Versioning](http://semver.org/).

## **[SynchDB 1.1](https://github.com/Hornetlabs/synchdb/releases/tag/v1.1) - 2025-04-17**

This release introduces support for the Oracle connector, along with robust data type transformation capabilities tailored for Oracle sources. SynchDB's core data processing engine has been significantly improved to handle complex and diverse combinations of data transformations more reliably.

Performance has also been enhanced across multiple areas. Redundant iterations and catalog lookups have been eliminated through intelligent caching, and JSON parsing performance has been significantly improved by adopting more efficient PostgreSQL JSON parser APIs. These enhancements reduce overhead and contribute to faster, more scalable replication.

SynchDB now fully supports PostgreSQL 16, 17 and IvorySQL 4.4, ensuring compatibility with the latest PostgreSQL-based platforms. In addition, the process for defining custom transformation rules has been refined with the integration of an object mapping management utility, offering a more intuitive and maintainable way to configure and extend transformation logic.

### **Added**

* added Oracle connector support.
* added support for PostgreSQL 16, 17 and IvorySQL 4.4
* added support for Oracle's variable size/scale data type `NUMBER` processing. 
* added a new GUC `synchdb.error_handling_strategy` to control the strategy to handle an error: can be skip, exit or retry.
* added a new GUC `synchdb.dbz_log_level to control` the log level of log4j used by Debezium runner engine.
* added a new table `synchdb_attribute` that stores all the remote table attribute information that synchdb is currently capturing.
* added a new view `synchdb_att_view` that shows a side-by-side comparison of data type, table and column name mappings.
* added a new tabled `synchdb_objmap` to store differet object mapping entries.
* added a new SQL function `synchdb_add_objamp()` to add new object mapping entries. It supports table name, column name, data type and transform expressions.
* added a new SQL function `synchdb_reload_objmap()` to force a connector to reload object mapping entries.
* added a new snapshot mode called `schemasync` in which the connector will obtain schemas info from remote database, create the synchronized tables and switch to paused state
* added `synchdb_add_extra_conninfo()` function to configure extra SSL related connection parameters to a connector.
* added `synchdb_del_extra_conninfo()` function to remove all extra parameters created by `synchdb_add_extra_conninfo()`.
* added `synchdb_del_conninfo()` function to remove an existing connector info, delete all the meta data and object mappings associated and shut it down as needed.
* added `synchdb_del_objmap()` to disable and remove a object mapping entry.

### **Changed**

* when processing ALTER TABLE change events, synchdb will only adds a primary key when the table itself has no primary key.
* removed the ability to configure a connector in `synchdb_add_conninfo()` to connect to a different destination PostgreSQL database other than the one where synchdb is installed.
* at the end of every DDL change event processing, synchdb will now make an update on the new `synchdb_attribute` table.
* normalized all the data type mapping entries to use lower case letters.
* a connector will correct the table name, column name and data type when the configured object mapping differs from current values upon starting or reloading.
* `rulefile` has been replaced by `synchdb_add_objmap()` utilities.
* All ID-based SQL function column definitions have their data types changed from `TEXT` to `NAME`.
* removed the `rulefile` parameter from `synchdb_add_conninfo()`.
* optimize DML parser to use cached hash table as cache rather than accessing the catalog and rebuilding the hash at every change event.
* DML parser now uses more efficient JSONB API calls for increased performance.
 * added source timestamp, dbz ttimestamp and pg timestamp of last batch's first and last change event in `synchdb_get_stats`. These represent the timing between data generation, dbz processing and pg processing.
 * optimize JNI calls to use cached JNI handle rather than recreating at every change event.
 * enhanced the way Synchdb marks a batch as complete by just marking the first and last change event within a batch, rather than all events.
 * revamped the data processing engine to be more modular in design so it can handle more complex data type mappings.
 * non-native data types are now processed based on their category rather than treating them all as texts.

### **Fixed**

* fixed an issue where SPI update or delete could fail to update or delete by including only the primary key fields in WHERE clause.
* fixed an issue with ALTER TABLE to error out when attempting to add a duplicate primary key.
* fixed an issue where synchdb occasionally incorrectly looks up a column's schema value from the json change event, causing subsequent processing to fail.
* fixed an issue in 'skip on error' mode to panic due to error stack overflow.
* fixed an issue in ALTER TABLE ADD and DROP column where it was not aware that the column name may have be mapped to a different value in PostgreSQL.

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
