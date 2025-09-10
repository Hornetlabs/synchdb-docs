# 配置 Infinispan 的 Oracle 连接器

## **概述**

默认情况下，Debezium 的 Oracle 连接器使用 JVM 堆缓存传入的变更事件，然后再将它们传递给 SynchDB 进行处理。最大堆大小通过 `synchdb.jvm_max_heap_size` GUC 控制。当 JVM 堆内存不足时（尤其是在处理大型事务或包含大量列的模式时），连接器可能会出现 OutOfMemoryError 错误，这使得堆大小调整成为一项关键的调优挑战。

作为替代方案，Debezium 可以配置使用 Infinispan 作为其缓存层。Infinispan 支持 JVM 堆和堆外（直接内存）的内存存储，从而提供更大的灵活性。此外，它还支持钝化功能，允许在达到内存限制时将多余的数据溢出到磁盘。这使得它在高负载下更具弹性，并确保能够优雅地处理大型事务或模式变更，而不会耗尽内存。

## **`synchdb_add_infinispan()`**

**签名：**

```sql
synchdb_add_infinispan(
	connectorName NAME, 		-- 连接器名称
	memoryType NAME, 			-- 内存类型，可以是“堆”或“非堆”
	memorySize INT 				-- 预留为缓存的内存大小（以 MB 为单位）
)
```

为给定的连接器注册一个由 Infinispan 支持的缓存配置。这允许连接器使用 Infinispan 缓冲更改事件，并支持基于堆和堆外内存分配以及溢出到磁盘（钝化）。

**注意：**

- 如果在当前正在运行的连接器上调用，则该设置将在下次重启时生效。
- 如果连接器已存在 Infinispan 缓存，它将被替换。
- memoryType='off_heap' 使用本机（直接）内存，不受 JVM 堆限制，但应谨慎调整大小。
- 当内存填满时，缓存将自动支持钝化到磁盘。

**示例：**
``` sql
SELECT synchdb_add_infinispan('oracleconn', 'off_heap', 2048);

```

## **`synchdb_del_infinispan()`**

**签名：**

```sql
synchdb_add_infinispan(
	connectorName NAME 			-- 连接器名称
)
```

删除指定连接器的 Infinispan 缓存配置及其相关的磁盘元数据。此操作将删除：

- 所有缓存文件
- 所有钝化（溢出到磁盘）状态
- 相关的 Infinispan 配置

**注意：**

- 此函数只能在连接器停止时执行。
- 在连接器处于活动状态时尝试运行此函数将导致错误。
- 永久禁用或重新配置连接器的缓存后端后，使用此命令进行清理。

**Example:**
``` sql
SELECT synchdb_del_infinispan('oracleconn');

```

## **`钝化`**

钝化简单来说就是，如果缓存已满，infinispan 会弹出并将数据写入磁盘。只要内存中有足够的空间来保存更改事件，就不会发生磁盘写入。数据将写入 `$PGDATA/pg_synchdb/ispn_[连接器名称]_[目标数据库名称]`

## **用于测试 Infinispan 大型事务的 Oracle SQL 示例**

**在 Oracle 中创建测试表：**

```sql
CREATE TABLE big_tx_test (
    id       NUMBER PRIMARY KEY,
    payload  VARCHAR2(4000),
    created  DATE DEFAULT SYSDATE
);

```

**创建一个相对较大的交易：**
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

**大事务下的行为**

1. 未设置 infinispan + 低 JVM heap size (128)
--> 将发生 OutOfMemory 错误

2. 低 JVM heap size (128) + 高 infinispan heap size (2048)
--> 大型事务成功处理

3. 低 JVM heap size (128) + 低 infinispan heap size (128)
--> 将发生钝化
--> `ispn_[连接器名称]_[目标数据库名称]` 大小在处理过程中会增加，大型事务处理完成后会减小
--> 大型事务已成功处理，但速度较慢。

4. 低 JVM heap size (128) + 高 infinispan heap size (2048)
--> 将发生 OutOfMemory 错误
--> 请勿将 infinispan 配置 heap size 超过 JVM 最大 heap 内存

5. 较低的 JVM heap size (128) + 较低的 infinispan heap size (64)
--> 将发生钝化
--> `ispn_[连接器名称]_[目标数据库名称]` 的大小将在处理过程中增加，并在大型事务处理完成后减小。
--> 不建议使用，因为 infinispan 可能会占用一半的 JVM heap，而剩余空间可能不足以用于其他 Debezium 操作。

6. 低 JVM heap size (128) + 相同的 infinispan heap size (128)
--> 不建议使用，因为 infinispan 可能会占用所有 JVM 堆空间，最终导致 OutOfMemory 错误。

7. 高 JVM heap size (2048) + 低 infinispan heap size (128)
--> 将发生钝化。
--> `ispn_[连接器名称]_[目标数据库名称]` 的大小在处理过程中会增加，并在大型事务处理完成后减小。
--> 大型事务处理成功。
--> 效率低下 - 因为只有一小部分 JVM heap 空间用作缓存 + 不必要的钝化。