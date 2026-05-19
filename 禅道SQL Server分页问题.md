---
title: 禅道SQL Server分页问题
date: 2022-04-08 23:09:00
tags:
  - 禅道
  - SQL Server
  - SQL
keywords: 禅道,SQL Server,LIMIT,OFFSET FETCH,分页
categories: 实操
---



# 禅道SQL Server分页问题

禅道从 MySQL 切到 SQL Server 后，最先炸的是分页。

MySQL 里随手写：

```sql
SELECT id, name
FROM zt_product
ORDER BY id DESC
LIMIT 20, 10;
```

SQL Server 里不认这个写法。

当时页面报错大概就是 SQL 语法错误，查日志才看到 `LIMIT`。

## MySQL 的 LIMIT

MySQL 常见两种：

```sql
-- 取前10条
SELECT * FROM zt_product LIMIT 10;

-- 跳过20条，再取10条
SELECT * FROM zt_product LIMIT 20, 10;
```

第二个里面：

```text
20 是 offset
10 是 page size
```

## SQL Server 取前几条

SQL Server 取前 10 条常用 `TOP`：

```sql
SELECT TOP (10) id, name
FROM dbo.zt_product
ORDER BY id DESC;
```

注意 `TOP` 是放在 `SELECT` 后面，不是 SQL 最后。

## SQL Server 分页

SQL Server 2012 以后可以用 `OFFSET FETCH`。

```sql
SELECT id, name
FROM dbo.zt_product
ORDER BY id DESC
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

这个比较像 MySQL 的：

```sql
LIMIT 20, 10
```

但 SQL Server 这里有个很重要的点：分页必须要有稳定的 `ORDER BY`。

不要写这种：

```sql
SELECT id, name
FROM dbo.zt_product
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

会直接不对。

## 实际改写例子

原 MySQL：

```sql
SELECT id, name, code, status
FROM zt_project
WHERE deleted = '0'
ORDER BY id DESC
LIMIT 0, 20;
```

SQL Server：

```sql
SELECT id, name, code, status
FROM dbo.zt_project
WHERE deleted = '0'
ORDER BY id DESC
OFFSET 0 ROWS FETCH NEXT 20 ROWS ONLY;
```

第二页：

```sql
SELECT id, name, code, status
FROM dbo.zt_project
WHERE deleted = '0'
ORDER BY id DESC
OFFSET 20 ROWS FETCH NEXT 20 ROWS ONLY;
```

## 没有排序字段怎么办

有些列表原来 MySQL 写得比较随意，没有明显排序。SQL Server 分页时必须补一个排序字段。

一般优先用主键：

```sql
ORDER BY id DESC
```

如果业务要求按创建时间：

```sql
ORDER BY openedDate DESC, id DESC
```

这里最好带上 `id` 做二级排序，不然创建时间相同的时候，翻页可能出现数据顺序飘。

## 性能问题

`OFFSET` 很大时性能会变差。比如：

```sql
OFFSET 100000 ROWS FETCH NEXT 20 ROWS ONLY
```

这种要看执行计划，至少排序字段上要有合适索引。

例如列表常按 `deleted + id` 查：

```sql
CREATE NONCLUSTERED INDEX IX_zt_project_deleted_id
ON dbo.zt_project(deleted, id DESC)
INCLUDE(name, code, status);
```

不是所有表都这么加，这里只是说明思路。要看真实查询条件。

## 当时记下来的坑

1. `LIMIT` 不能直接搬到 SQL Server。
2. `OFFSET FETCH` 必须配 `ORDER BY`。
3. 排序字段要稳定，最好带主键兜底。
4. 大页码分页要看索引，不然越翻越慢。
5. SQL Server 的对象名最好带 schema，比如 `dbo.zt_project`。

## 参考

* SELECT - ORDER BY / OFFSET FETCH：https://learn.microsoft.com/en-us/sql/t-sql/queries/select-order-by-clause-transact-sql
* TOP 文档：https://learn.microsoft.com/en-us/sql/t-sql/queries/top-transact-sql
