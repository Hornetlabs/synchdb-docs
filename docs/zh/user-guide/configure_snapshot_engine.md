# 設定快照引擎

## **初始快照與變更資料擷取 (CDC)**

初始快照是指將表格結構和初始資料從遠端資料庫遷移到 SynchDB 的過程。對於大多數連接器類型，此操作僅在首次啟動連接器時透過嵌入式 Debezium 運行器執行一次，因為只有 Debezium 引擎才能執行初始快照。初始快照完成後，變更資料擷取將開始將即時變更串流傳輸到 SynchDB。 「快照模式」可以進一步控制初始快照和 CDC 的行為。更多信息，請參閱[此處](https://docs.synchdb.com/user-guide/start_stop_connector/)。

除了基於 Debezium 的初始快照（當表數量眾多時可能會很慢）之外，SynchdB 還提供了另一個基於 FDW 的原生初始快照引擎。目前，只有原生 Openlog Replicator (OLR) 連接器支援基於 FDW 的快照。所有其他連接器仍然依賴 Debezium 來執行快照。

## **基於 FDW 的初始快照**

* 在 `postgresql.conf` 中將 "synchdb.olr_snapshot_engine" 設為 "fdw" 即可啟用。
* 啟動需要快照的 OLR 連接器，並設定為快照模式。例如：

```sql
SELECT synchdb_start_engine_bgw('olrconn', 'no_data');

-- 或
SELECT synchdb_start_engine_bgw('olrconn', 'always');

-- 或
SELECT synchdb_start_engine_bgw('olrconn', 'schemasync');

```

## **编译並安裝 oracle_fdw**

如果選擇 "fdw" 作為快照引擎，您需要確保相應的外部資料包裝器已安裝或可用於 synchdb。以下是從原始碼建置並安裝 oracle_fdw 的簡要步驟供您參考。

### **安裝 OCI**

建置和運行 oracle_fdw 需要 OCI。在此範例中，我使用的是 23.9.0 版本：

```bash
# 取得預編譯軟體包：
wget https://download.oracle.com/otn_software/linux/instantclient/2390000/instantclient-basic-linux.x64-23.9.0.25.07.zip
wget https://download.oracle.com/otn_software/linux/instantclient/2390000/instantclient-sdk-linux.x64-23.9.0.25.07.zip

# 解壓縮：選擇「Y」以覆寫元資料文件
unzip instantclient-basic-linux.x64-23.9.0.25.07.zip
unzip instantclient-sdk-linux.x64-23.9.0.25.07.zip
```

更新環境變量，以便系統知道在哪裡找到 OCI 頭檔和庫。

```bash
export OCI_HOME=/home/$USER/instantclient_23_9
export OCI_LIB_DIR=$OCI_HOME
export OCI_INC_DIR=$OCI_HOME/sdk/include
export LD_LIBRARY_PATH=$OCI_HOME:${LD_LIBRARY_PATH}
export PATH=$OCI_HOME:$PATH

```

您可以將上述命令新增至 `~/.bashrc` 的末尾，以便每次登入系統時自動設定 PATH。

您也可以將 $OCI_HOME 新增至 `/etc/ld.so.conf.d/x86_64-linux-gnu.conf`（以 Ubuntu 為例），作為尋找共用程式庫的新路徑。

您應該執行 ldconfig 命令，以便連結器知道在哪裡找到共用程式庫：

### **建置 oracle_fdw 2.8.0**

```bash
git clone https://github.com/laurenz/oracle_fdw.git --branch ORACLE_FDW_2_8_0

```

透過調整 Makefile 中的以下兩行，確保 oracle_fdw 的 Makefile 能夠找到 OCI 包含檔案和函式庫：

```bash
FIND_INCLUDE := $(wildcard /usr/include/oracle/*/client64 /usr/include/oracle/*/client /home/$USER/instantclient_23_9/sdk/include)
FIND_LIBDIRS := $(wildcard /usr/lib/oracle/*/client64/lib /usr/lib/oracle/*/client/lib /home/$USER/instantclient_23_9)

```

建置並安裝 oracle_fdw：

```bash
make PG_CONFIG=/usr/local/pgsql/bin/pg_config
sudo make install PG_CONFIG=/usr/local/pgsql/bin/pg_config

```

Oralce_fdw 已準備就緒。

**<<重要>>** 在使用基於 FDW 的初始快照之前，您無需執行“CREATE EXTENSION oracle_fdw”，也無需執行“CREATE SERVER”或“CREATE USER MAPPING”。 SynchDB 在執行快照時會處理所有這些操作。