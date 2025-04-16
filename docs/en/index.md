# About

<p align="center">
  <img src="images/synchdblogo.png" alt="synchdb" width="400">
</p>

## **About SynchDB**

SynchDB is a PostgreSQL extension that enables fast and reliable replication from heterogeneous databases — such as MySQL, SQL Server, and Oracle — directly into PostgreSQL. Unlike traditional data pipelines, SynchDB handles the entire synchronization and data conversion process natively within PostgreSQL, without requiring any middleware or external orchestration tools. It mainly manages the following end-to-end tasks:

* Establishes and maintains connections to external databases
* Captures change events from source systems
* Transforms these events into PostgreSQL-compatible formats
* Applies them to PostgreSQL

At its core, SynchDB integrates the Debezium Embedded Engine, a powerful Java-based change data capture (CDC) library that supports multiple database connectors. SynchDB bridges PostgreSQL's C-based runtime and Debezium's Java environment using the Java Native Interface (JNI), enabling seamless cooperation between both worlds.

This architecture allows PostgreSQL to leverage the rich ecosystem of Debezium connectors while keeping the extension lightweight, flexible, and easy to deploy.

🔗 Learn more about Debezium [here](https://debezium.io/documentation/reference/stable/index.html).

## **Notable Features**

- Efficient data synchronization
- Support replication from MySQL, SQL Server and Oracle databases
- Flexible data transformation rules, including table names, column names, data types and custom data transform expression
- Easy integration with existing PostgreSQL databases
- Initial snapshot and Change Data Capture (CDC) modes
- Support DDL and DML logical replication
- Global connector state, error and statistic provisioning

## **System Requirement**
- PostgreSQL 16 or 17
- Java Runetime Envirnment（JRE) 17 or above

## **Version History**

- [SynchDB v1.1](https://github.com/Hornetlabs/synchdb/releases/tag/v1.1)
- [SynchDB v1.0](https://github.com/Hornetlabs/synchdb/releases/tag/v1.0)
- [SynchDB v1.0 Beta1](https://github.com/Hornetlabs/synchdb/releases/tag/v1.0_beta1)

## **Getting Started**

Learn more about SynchDB architecture decisions [here](architecture/archiitecture/) or check out the [Installation Guide](user-guide/installation/) to get started with SynchDB.
