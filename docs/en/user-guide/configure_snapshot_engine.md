# Configure Snapshot Engine

## **Initial Snapshot vs Change Data Capture (CDC)**

Initial snapshot refers to the process of migrating both the table schema and the initial data from remote database to SynchDB. For most connector types, this is done only once on first connector start via embedded Debezium runner as the only engine that can perform initial snapshot. After the initial snapshot completes, the Change Data Capture will begin to stream live changes to SynchDB. The "snapshot modes" can further control the behavior of initial snapshot and CDC. Refer to [here](../../user-guide/start_stop_connector/) for more information.

In addition to Debezium based initial snapshot, which may be slow when there is a huge number of tables, SynchdB provides an alternative FDW-based native initial snapshot engine. FDW based snapshot is only supported in native Openlog Replicator (OLR) connector as of now. All other connectors still rely on debezium to carry out the snapshot.


## **FDW Based Initial Snapshot**

* Enabled in `postgresql.conf` by setting "synchdb.olr_snapshot_engine" to "fdw"
* Start a OLR connector with a snapshot mode that requires doing a snapshot. For example:

```sql
SELECT synchdb_start_engine_bgw('olrconn', 'no_data');

-- or
SELECT synchdb_start_engine_bgw('olrconn', 'always');

-- or
SELECT synchdb_start_engine_bgw('olrconn', 'schemasync');

```

## **Build and Install oracle_fdw**

If "fdw" is selected as snapshot engine, you need to ensure the corresponding Foreign Data Wrapper has been installed or available for synchdb. For your reference, below is a brief procedure to build and install oracle_fdw from source code.

### **Install OCI**

OCI is required to build and run oracle_fdw. For this example, I am using version 23.9.0:

```bash
# Get the pre-build packages:
wget https://download.oracle.com/otn_software/linux/instantclient/2390000/instantclient-basic-linux.x64-23.9.0.25.07.zip
wget https://download.oracle.com/otn_software/linux/instantclient/2390000/instantclient-sdk-linux.x64-23.9.0.25.07.zip

# unzip them: Select 'Y' to overwrite metadata files
unzip instantclient-basic-linux.x64-23.9.0.25.07.zip
unzip instantclient-sdk-linux.x64-23.9.0.25.07.zip
```

Update environment variables so the system knows where to find OCI headers and libraries.

```bash
export OCI_HOME=/home/$USER/instantclient_23_9
export OCI_LIB_DIR=$OCI_HOME
export OCI_INC_DIR=$OCI_HOME/sdk/include
export LD_LIBRARY_PATH=$OCI_HOME:${LD_LIBRARY_PATH}
export PATH=$OCI_HOME:$PATH

```

You could add the above commands to the end of `~/.bashrc` to automatically set the PATHs every time you login to your system.

You can also add $OCI_HOME to `/etc/ld.so.conf.d/x86_64-linux-gnu.conf` (ubuntu example), as a new path to find shared libraries.

You should do a ldconfig so your linker knows where to find shared libraries:

### **Build oracle_fdw 2.8.0**

```bash
git clone https://github.com/laurenz/oracle_fdw.git --branch ORACLE_FDW_2_8_0

```

Ensure oracle_fdw's Makefile can find OCI includes and libraries by adjusting these 2 lines in Makefile:

```bash
FIND_INCLUDE := $(wildcard /usr/include/oracle/*/client64 /usr/include/oracle/*/client /home/$USER/instantclient_23_9/sdk/include)
FIND_LIBDIRS := $(wildcard /usr/lib/oracle/*/client64/lib /usr/lib/oracle/*/client/lib /home/$USER/instantclient_23_9)

```

Build and install oracle_fdw:

```bash
make PG_CONFIG=/usr/local/pgsql/bin/pg_config
sudo make install PG_CONFIG=/usr/local/pgsql/bin/pg_config

```

Oralce_fdw is ready to go. 

**<<IMPORTANT>>** You do not have to run `CREATE EXTENSION oracle_fdw` prior to using FDW based initial snapshot, nor do you have to `CREATE SERVER` or `CREATE USER MAPPING`. SynchDB takes care of all of these when it performs the snapshot..

