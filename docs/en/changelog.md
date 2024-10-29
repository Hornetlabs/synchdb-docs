# Change Log
All notable changes to this project will be documented in this file.
 
The format is based on [Keep a Changelog](http://keepachangelog.com/)
and this project adheres to [Semantic Versioning](http://semver.org/).
 
## [SynchDB 1.0 Beta1] - 2024-10-23
 
The first SynchDB software release that lays a robust foundation for seamless replication from heterogeneous databases to PostgreSQL
 
### Added
* logical replication from heterogeneous databases: (MySQL and SQLServer).
* [DDL replication](../user-guide/ddl_replication) (CREATE TABLE, DROP TABLE, ALTER TABLE ADD COLUMN, ALTER TABLE DROP COLUMN, ALTER TABLE ALTER COLUMN)
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
* 2 data apply modes (SPI, HeapAM API)
* several [utility functions](../user-guide/utility_functions) to perform connector operations: (start, stop, pause, resume).
 
### Changed
N/A

### Fixed
N/A