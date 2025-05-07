# 架构
## **总体架构图**

![img](/images/synchdbarch.jpg)

SynchDB 扩展包含两个代码空间（Java 和 C），JNI 位于两者之间作为协调员。

**Java**

* Debezium Runner
* 嵌入式 Debezium 引擎

**C**

* SynchDB 启动器
* SynchDB Worker
* 格式转换器
* 复制代理
* 表同步代理（待定）

### **Debezium Runner (Java)**
* 使用“嵌入式 Debezium 引擎”的主要 Java 驱动程序。
* 创建、控制、管理和配置“嵌入式 Debezium 引擎”。
* 包含由“C 端”组件通过“JNI”调用的例程。

### **嵌入式 Debezium 引擎 (Java)**
* 支持各种连接器实现，以便从各种数据库类型（例如 MySQL、Oracle、SQL Server 等）复制变更数据。
* 连接并维护与外部数据库的连接，并获取其变更事件。

### **SynchDB 启动器**
* 负责使用 PostgreSQL 的后台工作器 API 创建和销毁 SynchDB 工作器。
* 配置每个工作器的连接器类型、目标数据库 IP、端口等。

### **SynchDB 工作器**
* 初始化 Java 虚拟机环境（JVM) 且实例化 `Debezium Runner` 以从特定连接器类型复制变更。
* 通过 JNI 与 Debezium Runner 通信，以接收 JSON 格式的变更数据。
* 将 JSON 变更事件传输到 `Format Converter` 模块进行进一步处理。

### **格式转换器**
* 使用 PostgreSQL Jsonb API 解析 JSON 变更事件
* 根据用户定义的转换规则，将 DDL 变更详情转换为与 PostgreSQL 兼容的 SQL 查询。
* 根据列数据类型进行处理，将 DML 变更详情转换为与 PostgreSQL 兼容的数据表示形式。它会生成原始的 HeapTupleData，可以直接将其输入到 PostgreSQL 中的堆访问方法，以加快执行速度。

### **复制代理**
* 处理**格式转换器**的输出。
* **格式转换器**将生成 HeapTupleData 格式的输出，然后**复制代理**将调用 PostgreSQL 的堆访问方法例程来处理它们。
* 对于 DDL 查询，**复制代理**将调用 PostgreSQL 的 SPI 来处理它们。

### **表同步代理**
* 设计细节和实现尚不可用。待定
* 旨在提供一种更高效的替代方案来执行初始表同步。

## **Java 原生接口 (JNI)**
Java 原生接口 (JNI) 是一个框架，允许 Java 应用程序与用 C 或 C++ 等语言编写的原生代码进行交互。它使 Java 程序能够调用原生应用程序和库，并被这些应用程序和库调用，从而在 Java 的平台独立性和原生代码的性能优势之间架起了一座桥梁。JNI 通常用于集成平台特定的功能、优化应用程序中性能关键部分，或访问 Java 中不可用的旧库。它需要谨慎管理资源，因为它涉及在 Java 虚拟机 (JVM) 和原生环境之间切换，这可能会增加复杂性。

SynchDB 需要 JNI 在 Debezium 运行引擎和 SynchDB PostgreSQL 扩展之间交换资源。JNI 可在 Java 安装（例如 openjdk）中使用。