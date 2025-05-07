# 关于

<p align="center">
<img src="../images/synchdblogo.jpg" alt="synchdb" width="400">
</p>

## **关于 SynchDB**

SynchDB 是一个 PostgreSQL 扩展，支持从异构数据库（例如 MySQL、SQL Server 和 Oracle）直接快速可靠地复制到 PostgreSQL。与传统的数据管道不同，SynchDB 在 PostgreSQL 内部原生处理整个同步和数据转换过程，无需任何中间件或外部编排工具。它主要管理以下端到端任务：

* 建立并维护与外部数据库的连接
* 从源系统捕获变更事件
* 将这些事件转换为与 PostgreSQL 兼容的格式
* 将它们应用于 PostgreSQL

SynchDB 的核心集成了 Debezium 嵌入式引擎，这是一个基于 Java 的强大变更数据捕获 (CDC) 库，支持多种数据库连接器。 SynchDB 使用 Java 原生接口 (JNI) 连接 PostgreSQL 基于 C 语言的运行时和 Debezium 的 Java 环境，从而实现两者之间的无缝协作。

这种架构使 PostgreSQL 能够充分利用 Debezium 丰富的连接器生态系统，同时保持扩展的轻量级、灵活性和易于部署。

🔗 了解更多关于 Debezium 的信息，请点击此处 (https://debezium.io/documentation/reference/stable/index.html)。

## **主要特性**

- 高效的数据同步
- 支持从 MySQL、SQL Server 和 Oracle 数据库复制
- 灵活的数据转换规则，包括表名、列名、数据类型和自定义数据转换表达式
- 轻松与现有 PostgreSQL 数据库集成
- 初始快照和变更数据捕获 (CDC) 模式
- 支持 DDL 和 DML 逻辑复制
- 全局连接器状态、错误和统计信息配置

## **系统要求**
- PostgreSQL 16 或 17
- Java 运行时环境（JRE）17 或更高版本

## **版本历史**

- [SynchDB v1.1](https://github.com/Hornetlabs/synchdb/releases/tag/v1.1)
- [SynchDB v1.0](https://github.com/Hornetlabs/synchdb/releases/tag/v1.0)
- [SynchDB v1.0 Beta1](https://github.com/Hornetlabs/synchdb/releases/tag/v1.0_beta1)

## **获取已开始**

了解有关 SynchDB 架构决策的更多信息，请点击[此处](architecture/architecture/) 或查看[安装指南](user-guide/installation/) 以开始使用 SynchDB。