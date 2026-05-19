---
title: SQL Server Always On部署
date: 2022-04-12 23:04:00
tags:
  - SQL Server
  - Always On
  - Pacemaker
keywords: SQL Server Always On,Linux,Pacemaker,Availability Group,SQL Server 2019
categories: 实操
---



# SQL Server Always On部署

这次 SQL Server 是 Linux 机器，不是 Windows 上那套 WSFC。

所以 Always On 这里要先说明一下：Linux 下可用性组本身还是 SQL Server 的 AG，但集群管理走 Pacemaker。客户说“主从容灾”，不能直接按 MySQL 主从去理解。

这里记录核心步骤，具体环境里的主机名和 IP 都脱敏。

```text
db01  10.10.20.11
db02  10.10.20.12
```

## 开启 HADR

两台都执行：

```bash
/opt/mssql/bin/mssql-conf set hadr.hadrenabled 1
systemctl restart mssql-server
```

检查：

```sql
SELECT SERVERPROPERTY('IsHadrEnabled') AS IsHadrEnabled;
```

返回 `1` 才对。

## 创建 master key、证书和 endpoint 用户

db01：

```sql
USE master;
GO
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<MASTER_KEY_PASSWORD>';
GO

CREATE CERTIFICATE db01_cert
WITH SUBJECT = 'db01 certificate';
GO

BACKUP CERTIFICATE db01_cert
TO FILE = '/var/opt/mssql/data/db01_cert.cer';
GO
```

db02 同理创建自己的证书。然后互相复制 `.cer` 文件。

例如 db02 上导入 db01 的证书，并建出对应的 login：

```sql
USE master;
GO
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<MASTER_KEY_PASSWORD>';
GO

CREATE CERTIFICATE db01_cert
FROM FILE = '/var/opt/mssql/data/db01_cert.cer';
GO

CREATE LOGIN db01_login FROM CERTIFICATE db01_cert;
GO
```

证书这块最容易因为文件权限卡住。SQL Server 进程要能读。

```bash
chown mssql:mssql /var/opt/mssql/data/*.cer
chmod 600 /var/opt/mssql/data/*.cer
```

## 创建 endpoint

两台都创建 endpoint，端口这里用 5022。注意 endpoint 要在给对方 login 授权前创建。

```sql
CREATE ENDPOINT Hadr_endpoint
STATE = STARTED
AS TCP (LISTENER_PORT = 5022)
FOR DATABASE_MIRRORING (
    ROLE = ALL,
    AUTHENTICATION = CERTIFICATE db01_cert,
    ENCRYPTION = REQUIRED ALGORITHM AES
);
GO
```

如果是在 db02，就用 db02 自己的证书。

endpoint 建完以后，再授权对方 login 连接 endpoint。比如 db02 上给 db01：

```sql
GRANT CONNECT ON ENDPOINT::Hadr_endpoint TO db01_login;
GO
```

db01 上也要反过来给 db02 做同样的导入、建 login、授权。这个是双向的。

放行端口：

```bash
firewall-cmd --permanent --add-port=5022/tcp
firewall-cmd --reload
```

互测：

```bash
nc -vz db01 5022
nc -vz db02 5022
```

## 准备数据库

db01 上：

```sql
ALTER DATABASE zentao SET RECOVERY FULL;
GO

BACKUP DATABASE zentao
TO DISK = N'/data/backup/zentao_full_ag.bak'
WITH INIT, COMPRESSION, CHECKSUM;
GO

BACKUP LOG zentao
TO DISK = N'/data/backup/zentao_log_ag.trn'
WITH INIT, COMPRESSION, CHECKSUM;
GO
```

把备份复制到 db02，还原成 NORECOVERY：

```sql
RESTORE DATABASE zentao
FROM DISK = N'/data/backup/zentao_full_ag.bak'
WITH NORECOVERY,
MOVE N'zentao' TO N'/var/opt/mssql/data/zentao.mdf',
MOVE N'zentao_log' TO N'/var/opt/mssql/data/zentao_log.ldf';
GO

RESTORE LOG zentao
FROM DISK = N'/data/backup/zentao_log_ag.trn'
WITH NORECOVERY;
GO
```

## 创建可用性组

db01 上示例：

```sql
CREATE AVAILABILITY GROUP ag_zentao
WITH (CLUSTER_TYPE = EXTERNAL)
FOR DATABASE zentao
REPLICA ON
    N'db01' WITH (
        ENDPOINT_URL = N'tcp://db01:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE = EXTERNAL,
        SEEDING_MODE = MANUAL
    ),
    N'db02' WITH (
        ENDPOINT_URL = N'tcp://db02:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE = EXTERNAL,
        SEEDING_MODE = MANUAL
    );
GO
```

db02 上加入：

```sql
ALTER AVAILABILITY GROUP ag_zentao JOIN WITH (CLUSTER_TYPE = EXTERNAL);
GO
ALTER DATABASE zentao SET HADR AVAILABILITY GROUP = ag_zentao;
GO
```

## 看状态

```sql
SELECT
    ag.name,
    ar.replica_server_name,
    ars.role_desc,
    ars.synchronization_health_desc,
    ars.connected_state_desc
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id;
```

数据库同步：

```sql
SELECT
    DB_NAME(database_id) AS db_name,
    synchronization_state_desc,
    synchronization_health_desc,
    log_send_queue_size,
    redo_queue_size
FROM sys.dm_hadr_database_replica_states;
```

## Pacemaker

Linux 下自动故障转移要配 Pacemaker。这个地方不要只在 SQL Server 里建 AG 就以为完事。

大体步骤是：

```text
1. 安装 pacemaker、pcs、资源代理
2. 配置 hacluster 用户
3. pcs cluster auth
4. pcs cluster setup/start
5. 创建 AG 资源
6. 创建虚拟 IP 资源
7. 设置 colocation/order 约束
8. 做故障转移测试
```

具体命令要按客户环境版本走，不能照抄到所有机器。

## 这里最容易错的地方

1. Linux AG 不是 Windows WSFC，集群层要 Pacemaker。
2. 数据库必须 FULL 恢复模式，并且辅助副本要先还原到 `NORECOVERY`。
3. endpoint 端口 5022 要互通。
4. 证书文件权限经常会卡。
5. `CLUSTER_TYPE = EXTERNAL` 不要漏。
6. 应用不要连 db01/db02 固定 IP，后面要连 Listener / VIP。

## 参考

* SQL Server Linux 可用性组概述：https://learn.microsoft.com/en-us/sql/linux/sql-server-linux-availability-group-overview
* 配置 Linux 可用性组：https://learn.microsoft.com/en-us/sql/linux/sql-server-linux-availability-group-configure-ha
* Pacemaker 集群配置：https://learn.microsoft.com/en-us/sql/linux/sql-server-linux-availability-group-cluster-pacemaker
