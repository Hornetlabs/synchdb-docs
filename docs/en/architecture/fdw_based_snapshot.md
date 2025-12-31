# FDW Based Snapshot

## **Overview**

All the connectors support Debezium based initial snapshot, which migrates remote table schemas to PostgreSQL with or without the initial data (depending on the snapshot mode used). This works mostly, but may suffer from performance issues when there is a large number of tables to migrate and extra overhead is introduced via JNI calls. It is possible to achieve the same initial snapshot using a Foreign Data Wrapper (FDW) if source database can support `FLASHBACK` query, such as Oracle. `FLASHBACK` query can ensure a snapshot to migrate up to a certain `offset` (SCN in Oracle), ensuring read consistency up to a point in time, and then begin the CDC streaming from this point. 

Currently, we support oracle_fdw as an alternative to Debezium based snapshot for native Openlog Replicator (OLR) connector only. In the future, we may add support for other connectors if feasible. To use FDW based snapshot, simply set GUC parameter `synchdb.olr_snapshot_engin` to "fdw". Refer to [here](user-guide/configure_snapshot_engine/) for a quick guide to build and install oracle_fdw.


## **How does FDW Based Snapshot Work**

FDW based snapshot consists of about 10 steps:

### **1. Preparation**

This steps checks if `oracle_fdw` is installed and available, creates a `server` and `user mapping` based on the connector information created by `synchdb_add_conninfo` 

### **2. Create Oracle Object Views** 

This steps create numerous foreign tables in a separate schema (ex. `ora_obj`) in PostgreSQL. These foreign tables (when queried) connects to Oracle via oracle_fdw and obtains most (if not all) of the objects available on Oracle. Objects such as:

* tables
* columns
* keys
* indexes
* functions
* sequences
* views
* triggers
* ...etc

SynchDB does not need to account for every single object to complete an initial snapshot; It only accounts for `tables`, `columns` and `keys` objects to construct the tables in PostgreSQL. 

### **3. Create a Foreign Table to Fetch Curretn SCN**

This foreign table will connect to Oracle (when queried) and fetch the current SCN value. This value will serve as a `cut-off` point for the initial snapshot. New changes after this point will not be considered in the snapshot.

### **4. Create a List of Desired Foreign Tables**

The goal for this step is to create a new staging schema (ex. ora_stage), and create desired foreign tables for the snapshot based on:

* Oracle object views created from step 2
* SynchDB's `data->'snapshottable'` parameter in `synchdb_conninfo` -> this is a filter of tables that SynchDB needs to do a snapshot. All tables will be considered in snapshot if set to `null`.
* Extra data type mapping as described in `synchdb_objmap` 
* the cut-off SCN obtained from step 3

At the end of this step, the staging schema will contain foreign tables with `AS OF SCN xxx` attribute with their data types mapped according to SycnhDB's data type mapping rules. These foreign tables will return the data only up to the SCN set and still require oracle_fdw to connect to Oracle.

### **5. Materialize the Schema**

The goal for this step is to materialize (turn foreign tables into real PostgreSQL tables) only the table schemas created from step 4 and put them in a new destination schema (ex. dst_stage). So, at the end of the step, the destination schema will contain real tables with the same structure as those in staging schema.

### **6. Migrate Primary Key**

Based on oracle object views created from step 2, do `ALTER TABLE ADD PRIMARY KEY` commands on the materialized tables created from step 5 to add primary keys as needed. 

### **7. Apply Column Name Mappings**

Based on column name mappings as described in `synchdb_objmap` , do `ALTER TABLE RENAME COLUMN` commands on the materialized tables created from step 5 to change column names as needed. 

### **8. Migrate Data With Transforms**

The goal for This step is to migrate the table data from staging to destination schemas and perform any value transforms as described in `synchdb_objmap`, basically doing something like:

* remote query
* apply data transform
* local insert

### **9. Apply Table Name Mappings**

Once the data is migrated, SynchDB will apply table name mapping as described in `synchdb_objmap`. The reason it is done last is that table name mapping could potentially move the table to some other schema in which we do not wish to happen during materialization. 

### **10. Finalize Initial Snapshot**

The initial snapshot is considered done at this point. This step will do a cleanup as follows:

* drop server and user mapping created from step 1
* drop Oracle object view created from step 2
* drop staging schema created from step 4
* drop the foreign table to return current SCN created from step 3

The tables that reside in the destination schema created from step 5 is the final result of initial snapshot.



