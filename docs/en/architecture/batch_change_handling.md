# Batch Change Handling

## **Overview**
SynchDB periodically fetches a batch of change request from Debezium runner engine at a period of `synchdb.naptime` milliseconds (default 100). This batch of change request is then processed by SynchDB. If all the change requests within the batch have been processed successfully (parsed, transformed and applied to PostgreSQL), SynchDB will notify the Debezium runner engine that this batch has been completed. This signals Debezium runner to commit the offset up until the last successfully completed change record. With this mechanism in place, SynchDB is able to track each change record and instruct Debezium runner not to fetch an old change that has been processed before, or not to send a duplcate change record.

## **Batch Handling**
SynchDB processes a batch within one transaction. This means the change events inside a batch are either all or none processed. When all the changes have been successfully processed, SynchDB simply sends a message to Debezium runner engine to mark the batch as processed and completed. This action causes offsets to be committed and eventually flush to disk. An offset represents a logical location during a replication similar to the LSN (Log Seqeuence Number) in PostgreSQL.

![img](/images/synchdb-batch-new.jpg)


If a batch of changes are partially successful on PostgreSQL, it would cause the transaction to rollback and SynchDB will not notify Debezium runner about batch completion. When the connector is restarted, the same batch will resume processing again. 

## **Potential Conflicts**
If a batch is successfully processed by PostgreSQL, but encountered a power outage right before it sends a batch completion request to Debezium, it would cause SynchDB to receive the same batch request when it restarts, because Debezium thinks this batch has not been processed but in fact it has. This would cause the batch to fail due to duplicate data. 

This issue is logged as [issue 108](https://github.com/Hornetlabs/synchdb/issues/108) and is to be addressed in near future.