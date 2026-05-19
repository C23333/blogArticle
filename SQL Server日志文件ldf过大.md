---
title: SQL Server日志文件ldf过大
date: 2022-04-11 22:27:00
tags:
  - SQL Server
  - 日志
  - 备份
keywords: SQL Server,ldf,事务日志,BACKUP LOG,DBCC SQLPERF
categories: 实操
---



# SQL Server日志文件ldf过大

SQL Server 刚开始压测没多久，客户那边说磁盘空间掉得很快。

上去一看，`.ldf` 比 `.mdf` 还大。这个时候不能直接删 ldf，也不要上来就 shrink。

SQL Server 的 ldf 是事务日志，不是普通日志文件。

## 先看日志空间

```sql
DBCC SQLPERF(LOGSPACE);
```

也可以查数据库文件：

```sql
SELECT
    DB_NAME(database_id) AS db_name,
    type_desc,
    name,
    size * 8 / 1024 AS size_mb,
    physical_name
FROM sys.master_files
WHERE DB_NAME(database_id) = 'zentao';
```

## 看恢复模式

```sql
SELECT name, recovery_model_desc
FROM sys.databases
WHERE name = 'zentao';
```

如果是 FULL 恢复模式，但一直没有做日志备份，日志就不会按预期截断，文件会一直涨。

## 正确做日志备份

```sql
BACKUP LOG zentao
TO DISK = N'/data/backup/zentao_log_20220411_2200.trn'
WITH INIT, COMPRESSION, CHECKSUM, STATS = 10;
```

做完以后再看：

```sql
DBCC SQLPERF(LOGSPACE);
```

注意，日志备份通常会释放可复用空间，但物理 ldf 文件大小不一定马上变小。

## 临时收缩

如果磁盘已经很紧，可以在日志备份后临时收缩一次。

先查逻辑文件名：

```sql
SELECT name, type_desc
FROM sys.database_files;
```

然后：

```sql
USE zentao;
DBCC SHRINKFILE (N'zentao_log', 2048);
```

这里 `2048` 是 MB。

这个只是临时处理，不要把 shrink 当定时任务天天跑。天天 shrink，后面业务写入又增长，来回折腾更差。

## 看有没有大事务

有时候做了日志备份也不释放，可能有长事务。

```sql
DBCC OPENTRAN('zentao');
```

看当前请求：

```sql
SELECT
    r.session_id,
    r.status,
    r.command,
    r.wait_type,
    r.total_elapsed_time,
    t.text
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.database_id = DB_ID('zentao')
ORDER BY r.total_elapsed_time DESC;
```

## 备份策略

当时先按这个思路：

```text
每天凌晨完整备份
每4小时差异备份
每30分钟日志备份
备份文件同步到客户指定备份目录
每周至少做一次恢复验证
```

SQL Server Agent 作业可以做，Linux 上也可以用 shell + sqlcmd + cron。客户环境里最后看他们运维习惯。

用 sqlcmd 执行备份脚本示例：

```bash
/opt/mssql-tools/bin/sqlcmd -S 127.0.0.1 -U sa -P '******' -Q "BACKUP LOG zentao TO DISK = N'/data/backup/zentao_log_$(date +%Y%m%d_%H%M).trn' WITH INIT, COMPRESSION, CHECKSUM"
```

## 记一下

1. ldf 不是普通日志，不能删。
2. FULL 模式下要做日志备份。
3. shrink 只能当临时处理，不是长期方案。
4. 大事务会影响日志截断。
5. 备份策略一定要配恢复演练，不然只有备份没有意义。

## 参考

* 事务日志备份：https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/transaction-log-backups-sql-server
* DBCC SQLPERF：https://learn.microsoft.com/en-us/sql/t-sql/database-console-commands/dbcc-sqlperf-transact-sql
* DBCC SHRINKFILE：https://learn.microsoft.com/en-us/sql/t-sql/database-console-commands/dbcc-shrinkfile-transact-sql
