# Configure Infinispan for Oracle Connector

## **Overview**

By default, Debezium's Oracle connector uses the JVM heap to cache incoming change events before passing them to SynchDB for processing. The maximum heap size is controlled via the `synchdb.jvm_max_heap_size` GUC. When the JVM heap runs out of memory (especially under large transactions or schemas with many columns), the connector may fail with OutOfMemoryError, making heap sizing a critical tuning challenge.

As an alternative, Debezium can be configured to use Infinispan as its caching layer. Infinispan supports memory storage on both the JVM heap and off-heap (direct memory), offering more flexibility. Additionally, it supports passivation, allowing excess data to be spilled to disk when memory limits are reached. This makes it far more resilient under heavy workloads and ensures large transactions or schema changes can be handled gracefully without running out of memory.

## **`synchdb_add_infinispan()`**

**Signature:**

```sql
synchdb_add_infinispan(
    connectorName NAME,     -- Name of the connector
    memoryType NAME,        -- memory type, can be "heap" or "off heap"
    memorySize INT          -- size of memory in MB to reserve as cache
)
```

Registers an Infinispan-backed caching configuration for a given connector. This allows the connector to use Infinispan for buffering change events, supporting both heap-based and off-heap memory allocation with spill-to-disk (passivation).

**Note:**

- If called on a connector that is currently running, the setting will take effect on next restart.
- If an Infinispan cache already exists for the connector, it will be replaced.
- memoryType='off_heap' utilizes native (direct) memory and is not limited by JVM heap, but should be sized carefully.
- The cache will automatically support passivation to disk when memory fills.

**Example:**
``` sql
SELECT synchdb_add_infinispan('oracleconn', 'off_heap', 2048);



```

## **`synchdb_del_infinispan()`**

**Signature:**

```sql
synchdb_add_infinispan(
    connectorName NAME         -- Name of the connector
)
```

Removes the Infinispan cache configuration and associated on-disk metadata for a given connector. This operation will delete:

- All cache files
- Any passivated (spilled-to-disk) state
- The associated Infinispan configuration

**Note:**

- This function can only be executed when the connector is stopped.
- Attempting to run it while the connector is active will result in an error.
- Use this command to clean up after permanently disabling or reconfiguring a connector’s cache backend.

**Example:**
``` sql
SELECT synchdb_del_infinispan('oracleconn');

```

## **`Passivation`**

Passivation simply means infinispan will eject and write data on disk if cache memory is full. As long as there is space in memory to hold change events, disk write will not occur. The data is written to `$PGDATA/pg_synchdb/ispn_[connector name]_[destination database name]`


## **Example Oracle SQLs to test Infinispan with a Large Transaction**

**Create a test table in Oracle:**
```sql
CREATE TABLE big_tx_test (
    id       NUMBER PRIMARY KEY,
    payload  VARCHAR2(4000),
    created  DATE DEFAULT SYSDATE
);

```

**Create a relatively large transaction:**
```sql
BEGIN
  FOR i IN 1..100000 LOOP
    INSERT INTO big_tx_test (id, payload)
    VALUES (i, RPAD('x', 4000, 'x'));
  END LOOP;
  COMMIT;
END;
/

```

**Behaviors Under Large Transaction**

1. no infinispan setting + low JVM heap (synchdb.jvm_max_heap_size=128)

--> OutOfMemory error will occur

2. low JVM heap (128) + high infinispan off heap (2048)

--> large transaction handled successfully

3. low JVM heap (128) + low infinispan off heap (128)

--> passivation will occur
--> `ispn_[connector name]_[destination database name]` size will grow during processing and reduce once large transaction finished processing
--> large transaction handled successfully but at slower speed.

4. low JVM heap (128) + high infinispan heap (2048)

--> OutOfMemory error will occur
--> do not configure infinispan to use more heap memory than JVM max heap.

5. low JVM heap (128) + low infinispan heap (64)

--> passivation will occur
--> `ispn_[connector name]_[destination database name]` size will grow during processing and reduce once large transaction finished processing
--> not recommended as half of the JVM heap can potentially be used by infinispan and may not have enough left for other Debezium operations

6. low JVM heap (128) + same infinispan heap (128)

--> not recommended as all of the JVM heap can potentially be used by infinispan and eventually causing OutOfMemory error

7. high JVM heap (2048) + low infinispan heap (128)

--> passivation will occur
--> `ispn_[connector name]_[destination database name]` size will grow during processing and reduce once large transaction finished processing
--> large transaction handled successfully
--> not efficient - as only a small portion of JVM heap is used as cache + unnessary passivation.