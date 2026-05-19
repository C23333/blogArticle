---
title: SQL Server差异备份还原不上
date: 2022-04-10 21:33:00
tags:
  - SQL Server
  - 备份
  - 还原
keywords: SQL Server,差异备份,NORECOVERY,RESTORE,备份链
categories: 实操
---



# SQL Server差异备份还原不上

客户问备份怎么做、能不能恢复到某个时间点。这个时候就不能按 MySQL dump 的思路想了。

SQL Server 这里有完整备份、差异备份、日志备份，顺序错了就还原不上。

当时差异备份还原失败，最后发现是基准备份不对，备份链断了。

## 先看恢复模式

```sql
SELECT name, recovery_model_desc
FROM sys.databases
WHERE name = 'zentao';
```

如果要做日志备份和时间点恢复，生产库一般是 FULL。

```sql
ALTER DATABASE zentao SET RECOVERY FULL;
```

注意：刚切到 FULL 后，要先做一次完整备份，后面的日志备份链才有意义。

## 完整备份

```sql
BACKUP DATABASE zentao
TO DISK = N'/data/backup/zentao_full_20220410.bak'
WITH INIT, COMPRESSION, CHECKSUM, STATS = 10;
```

## 差异备份

```sql
BACKUP DATABASE zentao
TO DISK = N'/data/backup/zentao_diff_20220410_1800.bak'
WITH DIFFERENTIAL, INIT, COMPRESSION, CHECKSUM, STATS = 10;
```

差异备份依赖最近一次完整备份。不是随便拿一个 full 都能配一个 diff。

## 日志备份

```sql
BACKUP LOG zentao
TO DISK = N'/data/backup/zentao_log_20220410_1830.trn'
WITH INIT, COMPRESSION, CHECKSUM, STATS = 10;
```

## 还原顺序

完整 + 差异：

```sql
RESTORE DATABASE zentao_restore
FROM DISK = N'/data/backup/zentao_full_20220410.bak'
WITH
    MOVE N'zentao' TO N'/var/opt/mssql/data/zentao_restore.mdf',
    MOVE N'zentao_log' TO N'/var/opt/mssql/data/zentao_restore_log.ldf',
    NORECOVERY,
    STATS = 10;

RESTORE DATABASE zentao_restore
FROM DISK = N'/data/backup/zentao_diff_20220410_1800.bak'
WITH RECOVERY, STATS = 10;
```

完整 + 差异 + 日志：

```sql
RESTORE DATABASE zentao_restore
FROM DISK = N'/data/backup/zentao_full_20220410.bak'
WITH
    MOVE N'zentao' TO N'/var/opt/mssql/data/zentao_restore.mdf',
    MOVE N'zentao_log' TO N'/var/opt/mssql/data/zentao_restore_log.ldf',
    NORECOVERY,
    STATS = 10;

RESTORE DATABASE zentao_restore
FROM DISK = N'/data/backup/zentao_diff_20220410_1800.bak'
WITH NORECOVERY, STATS = 10;

RESTORE LOG zentao_restore
FROM DISK = N'/data/backup/zentao_log_20220410_1830.trn'
WITH RECOVERY, STATS = 10;
```

中间步骤要 `NORECOVERY`，最后一步才 `RECOVERY`。

## 一直 Restoring 怎么办

如果数据库一直显示 Restoring，说明还没执行最后的 `WITH RECOVERY`。

确认不再继续还原日志后：

```sql
RESTORE DATABASE zentao_restore WITH RECOVERY;
```

但是注意，如果你还要继续追加日志备份，就不要提前 recovery。

## 看备份文件内容

```sql
RESTORE HEADERONLY
FROM DISK = N'/data/backup/zentao_diff_20220410_1800.bak';
```

看逻辑文件名：

```sql
RESTORE FILELISTONLY
FROM DISK = N'/data/backup/zentao_full_20220410.bak';
```

`MOVE` 里的逻辑文件名要用这里查出来的，不是随手写文件名。

## 这次问题点

差异备份还原不上，常见就是：

1. 用错完整备份。
2. 中间做过新的完整备份，差异基准变了。
3. 还原顺序错了。
4. 前一步用了 `RECOVERY`，后面不能继续还原。
5. `MOVE` 的逻辑文件名写错。

## 参考

* SQL Server 备份概述：https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/back-up-and-restore-of-sql-server-databases
* RESTORE 参数：https://learn.microsoft.com/en-us/sql/t-sql/statements/restore-statements-transact-sql
* 恢复模式：https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/recovery-models-sql-server
