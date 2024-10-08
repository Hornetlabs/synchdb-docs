---
title: 主页
---
# 主页

## 简介

SynchDB 是一个 PostgreSQL 扩展，旨在以快速可靠的方式将数据从一个或多个异构数据库（如 MySQL、MS SQLServer、Oracle 等）直接复制到 PostgreSQL。PostgreSQL 是多个异构数据库源的目标。无需中间件或第三方软件来协调异构数据库和 PostgreSQL 之间的数据同步。SynchDB 扩展本身能够处理所有数据同步需求。

它提供了两种可以使用内置 SQL 函数调用的关键工作模式：
* 同步模式（用于初始数据同步）
* 跟随模式（用于复制初始同步后的增量更改）

**同步模式**将表从异构数据库复制到 PostgreSQL，包括其架构、索引、触发器、其他表属性以及它所保存的当前数据。
**关注模式**订阅异构数据库中的表以获取增量更改并将其应用于 PostgreSQL 中的相同表，类似于 PostgreSQL 逻辑复制

## 功能

- 高效的数据同步
- 支持多个数据库源
- 轻松与现有 PostgreSQL 数据库集成

## 入门

查看我们的 [安装指南](user-guide/installation) 以开始使用 SynchDB。
