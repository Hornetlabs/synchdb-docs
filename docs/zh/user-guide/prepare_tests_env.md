# 准备异构数据库样本
此处提到的步骤旨在快速启动异构数据库，以便快速验证和功能演示。启动样本异构数据库所需的文件和脚本可以在 SynchDB 仓库 [这里](https://github.com/Hornetlabs/synchdb) 找到。

## 准备一个样本 MySQL 数据库
我们可以使用 docker compose 启动一个样本 MySQL 数据库进行测试。用户凭证在 `synchdb-mysql-test.yaml` 文件中描述。

```
docker compose -f synchdb-mysql-test.yaml up -d
```

以 `root` 身份登录 MySQL，并授予 `mysqluser` 用户执行实时 CDC 的权限
```
mysql -h 127.0.0.1 -u root -p

GRANT replication client on *.* to mysqluser;
GRANT replication slave  on *.* to mysqluser;
GRANT RELOAD ON *.* TO 'mysqluser'@'%';
FLUSH PRIVILEGES;
```

退出 mysql 客户端工具：
```
\q
```

## 准备一个样本 SQL Server 数据库
我们可以使用 docker compose 启动一个样本 SQL Server 数据库进行测试。用户凭证在 `synchdb-sqlserver-test.yaml` 文件中描述。
```
docker compose -f synchdb-sqlserver-test.yaml up -d
```
如果需要启用 SSL 证书的 SQL Server，请使用 synchdb-sqlserver-withssl-test.yaml 文件。

你可能没有安装 SQL Server 客户端工具，你可以登录到 SQL Server 容器来访问其客户端工具。

找出 SQL Server 的容器 ID：
```
id=$(docker ps | grep sqlserver | awk '{print $1}')
```

将数据库架构复制到 SQL Server 容器中：
```
docker cp inventory.sql $id:/
```

登录到 SQL Server 容器：
```
docker exec -it $id bash
```

根据架构构建数据库：
```
/opt/mssql-tools/bin/sqlcmd -U sa -P $SA_PASSWORD -i /inventory.sql
```

运行一些简单的查询（如果你使用的是启用 SSL 的 SQL Server，请添加 -N -C）：
```
/opt/mssql-tools/bin/sqlcmd -U sa -P $SA_PASSWORD -d testDB -Q "insert into orders(order_date, purchaser, quantity, product_id) values( '2024-01-01', 1003, 2, 107)"

/opt/mssql-tools/bin/sqlcmd -U sa -P $SA_PASSWORD -d testDB -Q "select * from orders"
```
