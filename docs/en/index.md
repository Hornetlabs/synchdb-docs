# Home

SynchDB is a PostgreSQL extension for synchronizing data from different database sources.

## Introduction

SynchDB is a PostgreSQL extension designed to replicate data from one or more heterogeneous databases (such as MySQL, MS SQLServer, Oracle, etc.) directly to PostgreSQL in a fast and reliable way. PostgreSQL serves as the destination from multiple heterogeneous database sources. No middleware or third-party software is required to orchestrate the data synchronization between heterogeneous databases and PostgreSQL. SynchDB extension itself is capable of handling all the data synchronization needs.

It provides two key work modes that can be invoked using the built-in SQL functions:
* Sync mode (for initial data synchronization)
* Follow mode (for replicate incremental changes after initial sync)

**Sync mode** copies tables from heterogeneous database into PostgreSQL, including its schema, indexes, triggers, other table properties as well as current data it holds.
**Follow mode** subscribes to tables in a heterogeneous database to obtain incremental changes and apply them to the same tables in PostgreSQL, similar to PostgreSQL logical replication

## Features

- Efficient data synchronization
- Support for multiple database sources
- Easy integration with existing PostgreSQL databases

## Getting Started

Check out our [Installation Guide](user-guide/installation.md) to get started with SynchDB.
