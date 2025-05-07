# Architecture
## **Overall Archtecture Diagram**

![img](/images/synchdbarch.jpg)

SynchDB extension consists of 2 code spaces (Java and C) with JNI sitting in between them as facilitator.

**Java**

* Debezium Runner
* Embedded Debezium Engine

**C**

* SynchDB Launcher
* SynchDB Worker
* Format Converter
* Replication Agent
* Table Synch Agent (To be determined)

### **Debezium Runner (Java)**
* The main java driver that utilizes the `Embedded Debezium Engine`.
* Creates, controls, manages and configures the `Embedded Debezium Engine`.
* Contains routines to be invoked by the components on the `C side` via `JNI`.

### **Embedded Debezium Engine (Java)**
* Supports various connector implementations to replicate change data from various database types such as MySQL, Oracle, SQL Server, etc.
* Connects and maintains connections with external databases and fetches their change events.

### **SynchDB Launcher**
* Responsible for creating and destroying SynchDB workers using PostgreSQL's background worker APIs.
* Configure each worker's connector type, destination database IPs, ports, etc.

### **SynchDB Worker**
* Initialize `Java Virtual Machine (JVM)` and instantiates a `Debezium Runner` to replicate changes from a specific connector type.
* Communicate with Debezium Runner via JNI to receive change data in JSON formats.
* Transfer the JSON change event to `Format Converter` module for further processing.

### **Format Converter**
* Parse the JSON change event using PostgreSQL Jsonb APIs
* Transform DDL change details to PostgreSQL compatible SQL queries following user-defined translation rules.
* Transform DML change details to PostgreSQL compatible data representation by processing them based on column data types. It Produces raw HeapTupleData which can be fed directly to Heap Access Method within PostgreSQL for faster executions.

### **Replication Agent**
* Processes the outputs from **`Format Converter`**.
* **`Format Converter`** will produce HeapTupleData format outputs, then **`Replication Agent`** will invoke PostgreSQL's heap access method routines to handle them.
* for DDL queries, **`Replication Agent`** will invoke PostgreSQL's SPI to handle them.

### **Table Sync Agent**
* Design details and implementation are not available yet. TBD
* Intended to provide a more efficient alternative to perform initial table synchronization.

## **Java Native Interface (JNI)**
Java Native Interface (JNI) is a framework that allows Java applications to interact with native code written in languages like C or C++. It enables Java programs to call and be called by native applications and libraries, providing a bridge between Java’s platform independence and the performance advantages of native code. JNI is commonly used to integrate platform-specific features, optimize performance-critical sections of an application, or access legacy libraries that are not available in Java. It requires careful management of resources, as it involves switching between the Java Virtual Machine (JVM) and native environments, which can introduce complexity.

SynchDB requires JNI to exchange resources between Debezium runner engine and SynchDB PostgreSQL extension. JNI is available with Java Installation (ex. openjdk).


