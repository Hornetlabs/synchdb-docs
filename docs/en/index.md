# About

![img](images/synchdblogo.png)

## About SynchDB

SynchDB is a PostgreSQL extension designed to replicate data from heterogeneous databases (such as MySQL, MS SQLServer and Oracle) directly to PostgreSQL in a fast and reliable way. SynchDB is responsible for initiating connection to these heterogeneous databases, obtaining change events, transforming them to PostgreSQL equivalent data strucutres and finally applying them in PostgreSQL. No middleware or third-party software is required to orchestrate this data synchronization.

SynchDB is powered by Java based Debezium Embedded engine which provides several types of connectors to interact with different heterogeneous database types. Learn more about Debezium [here](https://debezium.io/documentation/reference/stable/index.html).

## Notable Features

- Efficient data synchronization
- Support MySQL, SQL Server and Oracle databases
- Flexible data transformation rules
- Easy integration with existing PostgreSQL databases
- Initial snapshot and Change Data Capture (CDC) modes
- Support DDL and DML logical replication
- Global connector state, error and statistic provisioning

## System Requirement
- PostgreSQL 16 or 17
- Java Runetime Envirnment（JRE) 17 or above

## Version History

- [SynchDB v1.0](https://github.com/Hornetlabs/synchdb/releases/tag/v1.0)
- [SynchDB v1.0 Beta1](https://github.com/Hornetlabs/synchdb/releases/tag/v1.0_beta1)

## Getting Started

Check out our [Installation Guide](user-guide/installation/) to get started with SynchDB.
