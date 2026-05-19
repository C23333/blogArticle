---
title: 禅道接入SQL Server
date: 2022-04-05 23:18:00
tags:
  - 禅道
  - SQL Server
  - 数据库
keywords: 禅道,SQL Server,sqlcmd,CREATE LOGIN,SQL Server 2019
categories: 实操
---



# 禅道接入SQL Server

禅道默认更多是 MySQL 场景，这次客户要求后端数据库走 SQL Server。

一开始我以为就是改个连接配置，后来发现不只是连接问题。SQL Server 里登录名、数据库用户、schema、权限是分层的，和 MySQL 那种建个用户授权库的感觉不太一样。

这里先记最基础的接入过程。

## 版本

数据库机器是 CentOS 7，SQL Server 用的是 2019 15.x。

```bash
/opt/mssql-tools/bin/sqlcmd -S 127.0.0.1 -U sa -P '******' -Q "SELECT @@VERSION"
```

或者进 SQL 后查：

```sql
SELECT
    SERVERPROPERTY('ProductVersion') AS product_version,
    SERVERPROPERTY('ProductLevel') AS product_level,
    SERVERPROPERTY('Edition') AS edition;
```

## 建库

数据库名这里用 `zentao` 举例。

```sql
CREATE DATABASE zentao;
GO
```

看一下是否创建成功：

```sql
SELECT name, state_desc, recovery_model_desc
FROM sys.databases
WHERE name = 'zentao';
GO
```

如果是生产库，恢复模式一般会改成 FULL，后面配合备份链。

```sql
ALTER DATABASE zentao SET RECOVERY FULL;
GO
```

## 建登录名和用户

SQL Server 这里要分清楚：

* `LOGIN` 是实例级的，负责能不能登录 SQL Server
* `USER` 是数据库级的，负责进到某个库以后能不能查表、写表

先建 login：

```sql
CREATE LOGIN zentao_app
WITH PASSWORD = '<DB_PASSWORD>';
GO
```

再进业务库建 user：

```sql
USE zentao;
GO

CREATE USER zentao_app FOR LOGIN zentao_app;
GO
```

开发阶段为了先跑通，可能会直接给读写：

```sql
ALTER ROLE db_datareader ADD MEMBER zentao_app;
ALTER ROLE db_datawriter ADD MEMBER zentao_app;
GO
```

如果初始化要建表、改表，还需要 DDL 权限。这个不要长期保留，可以初始化后收回。

```sql
ALTER ROLE db_ddladmin ADD MEMBER zentao_app;
GO
```

初始化完成后撤掉：

```sql
ALTER ROLE db_ddladmin DROP MEMBER zentao_app;
GO
```

## 从应用机测试连接

先别急着让禅道连。应用机上先用 `sqlcmd` 测一下。

```bash
/opt/mssql-tools/bin/sqlcmd -S 10.10.20.11,1433 -U zentao_app -P '<DB_PASSWORD>' -d zentao -Q "SELECT DB_NAME(), SUSER_SNAME();"
```

能输出 `zentao` 和当前用户，再继续看 PHP。

如果这里都连不上，问题就在网络、端口、账号、密码、数据库权限，不要去改禅道代码。

## schema 问题

SQL Server 默认 schema 常见是 `dbo`。如果用户默认 schema 不对，建表时可能建到奇怪的 schema 下。

```sql
ALTER USER zentao_app WITH DEFAULT_SCHEMA = dbo;
GO
```

查一下：

```sql
SELECT name, default_schema_name
FROM sys.database_principals
WHERE name = 'zentao_app';
GO
```

## 简单建表验证

```sql
USE zentao;
GO

CREATE TABLE dbo.conn_test (
    id INT IDENTITY(1,1) PRIMARY KEY,
    name NVARCHAR(50) NOT NULL,
    created_at DATETIME2 NOT NULL DEFAULT SYSDATETIME()
);
GO

INSERT INTO dbo.conn_test(name) VALUES (N'禅道连接测试');
GO

SELECT * FROM dbo.conn_test;
GO
```

测完删掉：

```sql
DROP TABLE dbo.conn_test;
GO
```

## 当时踩到的点

1. login 能登录，不代表这个库里有 user。
2. user 有了，不代表有表权限。
3. 初始化阶段可能要 `db_ddladmin`，但跑完最好收掉。
4. SQL Server 字符串里中文建议用 `N'中文'`，不然后面字符集问题不好判断。
5. 连接测试先用 `sqlcmd`，不要一上来就从应用层猜。

## 参考

* CREATE LOGIN：https://learn.microsoft.com/en-us/sql/t-sql/statements/create-login-transact-sql
* CREATE USER：https://learn.microsoft.com/en-us/sql/t-sql/statements/create-user-transact-sql
* ALTER ROLE：https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-role-transact-sql
