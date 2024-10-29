# 架构
## 架构图
SynchDB 扩展由六个主要组件组成：

* Debezium 运行引擎（Java）
* SynchDB 启动器
* SynchDB 工作器
* 格式转换器
* 复制代理
* 表同步代理（待定）

请参考架构图以直观地了解组件及其交互。
![img](https://www.highgo.ca/wp-content/uploads/2024/07/synchdb.drawio.png)

### Debezium 运行引擎（Java）
* 一个利用 Debezium 嵌入式库的 Java 应用程序。
* 支持各种连接器实现，以从 MySQL、Oracle、SQL Server 等各种数据库类型复制变更数据。
* 由 `SynchDB 工作器` 调用，以初始化 Debezium 嵌入式库并接收变更数据。
* 以通用 JSON 格式将变更数据发送给 `SynchDB 工作器` 进行进一步处理。

### SynchDB 启动器
* 负责使用 PostgreSQL 的后台工作器 API 创建和销毁 SynchDB 工作器。
* 配置每个工作器的连接器类型、目标数据库 IP、端口等。

### SynchDB 工作器
* 实例化一个 `Debezium 运行引擎` 以从特定连接器类型复制变更。
* 通过 JNI 与 Debezium 运行器通信，以 JSON 格式接收变更数据。
* 将 JSON 变更数据传输到 `格式转换器` 模块进行进一步处理。

### 格式转换器
* 使用 PostgreSQL Jsonb API 解析 JSON 变更数据
* 根据用户定义的转换规则，将 DDL 变更详情转换为 PostgreSQL 兼容的 SQL 查询。
* 通过基于列数据类型处理 DML 变更详情，将其转换为 PostgreSQL 兼容的数据表示。它生成原始的 HeapTupleData，可以直接输入到 PostgreSQL 内的堆访问方法中，以实现更快的执行。

### 复制代理
* 处理 **`格式转换器`** 的输出。
* **`格式转换器`** 将生成 HeapTupleData 格式的输出，然后 **`复制代理`** 将调用 PostgreSQL 的堆访问方法例程来处理它们。
* 对于 DDL 查询，**`复制代理`** 将调用 PostgreSQL 的 SPI 来处理它们。

### 表同步代理
* 设计细节和实现尚未确定。待定
* 旨在提供一个更高效的替代方案来执行初始表同步。

## Java 本地接口（JNI）
Java 本地接口（JNI）是一个允许 Java 应用程序与用 C 或 C++ 等语言编写的本地代码交互的框架。它使 Java 程序能够调用和被本地应用程序和库调用，为 Java 的平台独立性和本地代码的性能优势提供了桥梁。JNI 通常用于集成特定平台的功能、优化应用程序的性能关键部分或访问 Java 中不可用的遗留库。它需要仔细管理资源，因为它涉及在 Java 虚拟机（JVM）和本地环境之间切换，这可能会引入复杂性。

SynchDB 需要 JNI 在 Debezium 运行引擎和 SynchDB PostgreSQL 扩展之间交换资源。JNI 随 Java 安装（例如 openjdk）提供。