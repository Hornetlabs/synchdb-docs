# Batch Change Handling

## Overview
SynchDB periodically fetches a batch of change request from Debezium runner engine at a period of `synchdb.naptime` milliseconds (default 500). This batch of change request is then processed by SynchDB. If all the change requests within the batch have been processed successfully (parsed, transformed and applied to PostgreSQL), SynchDB will notify the Debezium runner engine that this batch has been completed. This signals Debezium runner to commit the offset up until the last successfully completed change record. With this mechanism in place, SynchDB is able to track each change record and instruct Debezium runner not to fetch an old change that has been processed before, or not to send a duplcate change record.


## Batch Handling on Success
If all change requests inside a batch have all been successfully processed, SynchDB simply sends a message to Debezium runner engine to mark the batch as processed and completed, which caused offsets to be committed and eventually flush to disk.

![img](https://www.highgo.ca/wp-content/uploads/2024/10/synchdb-Page-4.drawio.png)

## Batch Handling on Partial Success
In the case when one of the change requests has failed to process due to internal PostgreSQL errors such as duplicate key violation, SynchDB will still notify Debezium runner engine about a partially completed batch. This notify message contains an indicator about the last successfully processed record. Debezium will then mark all the completed records as processed, while leaving unprocessed and failed records as they are. With this, when SynchDB and Debezium runner restart, they will resume from the last failed change record and will only proceed when the fault has been cleared.

![img](https://www.highgo.ca/wp-content/uploads/2024/10/synchdb-Page-5.drawio.png)
