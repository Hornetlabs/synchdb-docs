# Architecture
## Archtecture Diagram
SynchDB extension consists of six major components:

* Debezium Runner Engine (Java)
* SynchDB Launcher
* SynchDB Worker
* Format Converter
* Replication Agent
* Table Synch Agent (TBD)

Refer to the architecture diagram for a visual representation of the components and their interactions.
![img](https://www.highgo.ca/wp-content/uploads/2024/07/synchdb.drawio.png)

### Debezium Runner Engine (Java)
* A java application utilizing Debezium embedded library.
* Supports various connector implementations to replicate change data from various database types such as MySQL, Oracle, SQL Server, etc.
* Invoked by `SynchDB Worker` to initialize Debezium embedded library and receive change data.
* Send the change data to `SynchDB Worker` in generalized JSON format for further processing.

### SynchDB Launcher
* Responsible for creating and destroying SynchDB workers using PostgreSQL's background worker APIs.
* Configure each worker's connector type, destination database IPs, ports, etc.

### SynchDB Worker
* Instantiates a `Debezium Runner Engine` to replicate changes from a specific connector type.
* Communicate with Debezium Runner via JNI to receive change data in JSON formats.
* Transfer the JSON change data to `Format Converter` module for further processing.

### Format Converter
* Parse the JSON change data using PostgreSQL Jsonb APIs
* Transform DDL change details to PostgreSQL compatible SQL queries following user-defined translation rules.
* Transform DML change details to PostgreSQL compatible data representation by processing them based on column data types. It Produces raw HeapTupleData which can be fed directly to Heap Access Method within PostgreSQL for faster executions.

### Replication Agent
* Processes the outputs from **`Format Converter`**.
* **`Format Converter`** will produce HeapTupleData format outputs, then **`Replication Agent`** will invoke PostgreSQL's heap access method routines to handle them.
* for DDL queries, **`Replication Agent`** will invoke PostgreSQL's SPI to handle them.

### Table Sync Agent
* Design details and implementation are not available yet. TBD
* Intended to provide a more efficient alternative to perform initial table synchronization.

## Jave Native Interface (JNI)
Java Native Interface (JNI) is a framework that allows Java applications to interact with native code written in languages like C or C++. It enables Java programs to call and be called by native applications and libraries, providing a bridge between Java’s platform independence and the performance advantages of native code. JNI is commonly used to integrate platform-specific features, optimize performance-critical sections of an application, or access legacy libraries that are not available in Java. It requires careful management of resources, as it involves switching between the Java Virtual Machine (JVM) and native environments, which can introduce complexity.

SynchDB requires JNI to exchange resources between Debezium runner engine and SynchDB PostgreSQL extension. JNI is available with Java Installation (ex. openjdk).


