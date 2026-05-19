---
title: 禅道SQL Server自增ID问题
date: 2022-04-09 20:55:00
tags:
  - 禅道
  - SQL Server
  - SQL
keywords: SQL Server,IDENTITY,SCOPE_IDENTITY,自增ID,禅道
categories: 实操
---



# 禅道SQL Server自增ID问题

分页改完以后，又遇到一个问题：新增数据成功了，但后面拿自增 ID 的地方不对。

MySQL 里常见是 `AUTO_INCREMENT`，插入后用 `LAST_INSERT_ID()` 或驱动里的 lastInsertId。SQL Server 里是 `IDENTITY`，取法不一样。

## MySQL 写法

MySQL 建表大概这样：

```sql
CREATE TABLE zt_demo (
    id INT NOT NULL AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    PRIMARY KEY (id)
);
```

插入：

```sql
INSERT INTO zt_demo(name) VALUES ('test');
SELECT LAST_INSERT_ID();
```

## SQL Server 写法

SQL Server 自增列用 `IDENTITY`：

```sql
CREATE TABLE dbo.zt_demo (
    id INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    name NVARCHAR(50) NOT NULL
);
```

`IDENTITY(1,1)` 的意思是：

```text
从1开始，每次加1
```

插入：

```sql
INSERT INTO dbo.zt_demo(name) VALUES (N'test');
```

取刚插入的 ID：

```sql
SELECT SCOPE_IDENTITY() AS id;
```

## 不要乱用 @@IDENTITY

SQL Server 里还有：

```sql
SELECT @@IDENTITY;
```

但这个容易受触发器影响。比如插入 A 表时触发器又插入了 B 表，`@@IDENTITY` 可能拿到 B 表的 identity。

所以业务里一般用：

```sql
SELECT SCOPE_IDENTITY();
```

更稳一点。

## OUTPUT INSERTED.id

如果想在插入时直接拿 ID，也可以用 `OUTPUT`：

```sql
INSERT INTO dbo.zt_demo(name)
OUTPUT INSERTED.id
VALUES (N'test2');
```

批量插入时也能返回多行：

```sql
INSERT INTO dbo.zt_demo(name)
OUTPUT INSERTED.id, INSERTED.name
VALUES (N'a'), (N'b'), (N'c');
```

这个在改批量导入时比较好用。

## PDO 里要注意

如果 PHP 里原来直接依赖 MySQL 的 lastInsertId，切到 SQL Server 后要实际测一下。

当时我更愿意在 SQL 里显式补：

```sql
INSERT INTO dbo.zt_demo(name) VALUES (?);
SELECT SCOPE_IDENTITY() AS id;
```

然后在代码里取结果。

不要想当然认为所有 PDO 驱动表现一样。

## 临时插入指定 ID

有些初始化脚本需要插入固定 id。SQL Server 默认不允许直接插 identity 列，需要打开：

```sql
SET IDENTITY_INSERT dbo.zt_demo ON;

INSERT INTO dbo.zt_demo(id, name) VALUES (100, N'init');

SET IDENTITY_INSERT dbo.zt_demo OFF;
```

注意同一时间一个 session 只能对一张表开 `IDENTITY_INSERT`。

## 小结

这块改造时主要记住：

1. MySQL 是 `AUTO_INCREMENT`，SQL Server 是 `IDENTITY`。
2. SQL Server 取当前插入 ID 优先用 `SCOPE_IDENTITY()` 或 `OUTPUT INSERTED.id`。
3. 不要随便用 `@@IDENTITY`。
4. 初始化固定 ID 时才考虑 `IDENTITY_INSERT`。
5. 批量插入拿 ID，用 `OUTPUT` 比一条条查更清楚。

## 参考

* IDENTITY 属性：https://learn.microsoft.com/en-us/sql/t-sql/statements/create-table-transact-sql-identity-property
* SCOPE_IDENTITY：https://learn.microsoft.com/en-us/sql/t-sql/functions/scope-identity-transact-sql
* OUTPUT 子句：https://learn.microsoft.com/en-us/sql/t-sql/queries/output-clause-transact-sql
