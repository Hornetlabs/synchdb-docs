---
title: 主页
---
# 主页

SynchDB 是一个 PostgreSQL 扩展，能够从不同的数据库源实时捕获变更数据 (CDC)。

## 简介

SynchDB 是一个 PostgreSQL 扩展，旨在以快速可靠的方式将数据从一个或多个异构数据库（如 MySQL、MS SQLServer、Oracle 等）直接复制到 PostgreSQL。PostgreSQL 是多个异构数据库源的目标。无需中间件或第三方软件来协调异构数据库和 PostgreSQL 之间的数据同步。SynchDB 扩展本身能够处理所有数据同步需求。

## 显著功能

- 高效的数据同步
- 支持多个数据库源
- 灵活的数据转换规则
- 轻松与现有 PostgreSQL 数据库集成
- 初始快照（Initial Snapshot）和变更数据捕获 (CDC) 模式
- 支持 DDL 和 DML 逻辑复制
- 全局连接器状态、错误和统计信息查看

## 系统要求
- PostgreSQL 16
- Java Runetime Envirnment（JRE) 17 或以上

## 版本历史

- [SynchDB v1.0](https://github.com/Hornetlabs/synchdb/releases/tag/v1.0)
- [SynchDB v1.0 Beta1](https://github.com/Hornetlabs/synchdb/releases/tag/v1.0_beta1)

## 入门

查看我们的 [安装指南](user-guide/installation) 以开始使用 SynchDB。
