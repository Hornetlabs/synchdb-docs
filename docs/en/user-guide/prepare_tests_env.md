---
weight: 20
---
# Quick Test Environment
The procedures mentioned here are meant to quickly start up heterogeneous for quick verification and feature demostration. The necessary files and scripts for starting sample heterogeneous databases can be found at SynchDB repository [here](https://github.com/Hornetlabs/synchdb/testenv/)

## Prepare a Sample MySQL Database
We can start a sample MySQL database for testing using docker compose. The user credentials are described in the `synchdb-mysql-test.yaml` file
```
docker compose -f synchdb-mysql-test.yaml up -d
```

Login to MySQL as `root` and grant permissions to user `mysqluser` to perform real-time CDC
```sql
mysql -h 127.0.0.1 -u root -p

GRANT replication client on *.* to mysqluser;
GRANT replication slave  on *.* to mysqluser;
GRANT RELOAD ON *.* TO 'mysqluser'@'%';
FLUSH PRIVILEGES;
```

Exit mysql client tool:
```
\q
```

## Prepare a Sample SQL Server Database
We can start a sample SQL Server database for testing using docker compose. The user credentials are described in the `synchdb-sqlserver-test.yaml` file
```
docker compose -f synchdb-sqlserver-test.yaml up -d
```
use synchdb-sqlserver-withssl-test.yaml file for the SQL Server with SSL certificate enabled.

You may not have SQL Server client tool installed, you could login to SQL Server container to access its client tool.

Find out the container ID for SQL server:
```
id=$(docker ps | grep sqlserver | awk '{print $1}')
```

Copy the database schema into SQL Server container:
```
docker cp inventory.sql $id:/
```

Log in to SQL Server container:
```
docker exec -it $id bash
```

Build the database according to the schema:
```
/opt/mssql-tools/bin/sqlcmd -U sa -P $SA_PASSWORD -i /inventory.sql
```

Run some simple queries (add -N -C if you are using SSL enabled SQL Server):
```
/opt/mssql-tools/bin/sqlcmd -U sa -P $SA_PASSWORD -d testDB -Q "insert into orders(order_date, purchaser, quantity, product_id) values( '2024-01-01', 1003, 2, 107)"

/opt/mssql-tools/bin/sqlcmd -U sa -P $SA_PASSWORD -d testDB -Q "select * from orders"
```

## Prepare a Sample Oracle Database
We can use the free Oracle database docker image provided by Oracle for testing and evaluation of SynchDB. It comes with a free container database `FREE` and a pluggable database `FREEPDB1`
```
docker run -d -p 1521:1521 container-registry.oracle.com/database/free:latest
```

Find out the container ID and login to it:
```
id=$(docker ps | grep oracle | awk '{print $1}')
docker exec -it $id bash
```

Follow the procedure described [here](https://docs.synchdb.com/user-guide/remote_database_setups/) to set up logminer and logminer user