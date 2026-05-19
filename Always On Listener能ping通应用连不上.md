---
title: Always On Listener能ping通应用连不上
date: 2022-04-13 22:42:00
tags:
  - SQL Server
  - Always On
  - Listener
keywords: Always On Listener,SQL Server,应用连不上,连接串,Pacemaker
categories: 实操
---



# Always On Listener能ping通应用连不上

Always On 搭完以后，Listener / VIP 能 ping 通，但是禅道应用连不上。

这个现象很容易误导人。能 ping 通只能说明 IP 层大概通，不代表 SQL Server 1433、Listener、当前主副本、应用连接串都没问题。

## 先确认 VIP 在哪台

Linux + Pacemaker 场景，先看集群资源：

```bash
pcs status
```

看 VIP 当前落在哪台机器上。然后确认这台是不是 SQL Server 当前主副本。

SQL 查：

```sql
SELECT
    ar.replica_server_name,
    ars.role_desc,
    ars.synchronization_health_desc
FROM sys.availability_replicas ar
JOIN sys.dm_hadr_availability_replica_states ars
  ON ar.replica_id = ars.replica_id;
```

如果 VIP 在 db02，但主副本还在 db01，这就不对。

## ping 不等于 1433 通

应用机上测端口：

```bash
nc -vz 10.10.20.100 1433
```

这里 `10.10.20.100` 是脱敏后的 Listener/VIP。

如果不通，看：

```bash
firewall-cmd --list-all
ss -lntp | grep 1433
pcs status
```

SQL Server 机器上 1433 要监听，VIP 所在机器防火墙也要放行。

## 用 sqlcmd 连 Listener

应用机上不要只测 db01/db02，要测 Listener。

```bash
sqlcmd -S 10.10.20.100,1433 -U zentao_app -P '******' -d zentao -Q "SELECT @@SERVERNAME, DB_NAME();"
```

能连上以后，再看当前是不是主库：

```sql
SELECT
    DATABASEPROPERTYEX('zentao', 'Updateability') AS updateability;
```

如果是只读副本，应用写入肯定会出问题。

也可以试写一个测试表，测完删掉。

```sql
CREATE TABLE dbo.ag_write_test(id INT IDENTITY(1,1), name NVARCHAR(20));
INSERT INTO dbo.ag_write_test(name) VALUES(N'test');
SELECT * FROM dbo.ag_write_test;
DROP TABLE dbo.ag_write_test;
```

## 应用连接串

原来连 db01：

```text
10.10.20.11,1433
```

应该改成 Listener/VIP：

```text
10.10.20.100,1433
```

PHP PDO 示例：

```php
$dsn = "sqlsrv:Server=10.10.20.100,1433;Database=zentao;TrustServerCertificate=true";
$pdo = new PDO($dsn, "zentao_app", "******");
```

如果客户内部用了 DNS，也可以写域名：

```text
sql-zt-listener.intra.example,1433
```

但是 DNS 解析要查清楚：

```bash
nslookup sql-zt-listener.intra.example
dig sql-zt-listener.intra.example
```

## 切换后还连旧主库

如果应用连接串里写死 db01，那主备切换后肯定还去找旧主库。

这个问题不要靠“切完再改配置”解决，应该一开始就让应用连 Listener。

检查配置：

```bash
grep -R "10.10.20.11\|db01\|1433" -n /data/www/zentao/config 2>/dev/null
```

## 常见原因

1. VIP 能 ping，但 1433 没放行。
2. VIP 不在当前主副本所在机器。
3. 应用连接串还写 db01/db02。
4. DNS 解析到旧 IP。
5. SQL Server login 在主副本有，但切换后 SID/用户映射不一致。
6. 证书或加密参数导致 PHP 驱动连接失败。

## 最后检查顺序

```text
pcs status
-> SQL 当前主副本
-> VIP 所在机器
-> 应用机 nc 1433
-> sqlcmd 连 Listener
-> 测试写入
-> 应用连接串
-> PHP 驱动报错
```

不要被 ping 通骗了。

## 参考

* Linux 可用性组概述：https://learn.microsoft.com/en-us/sql/linux/sql-server-linux-availability-group-overview
* 可用性组监听器概念：https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/listeners-client-connectivity-application-failover
* SQL Server PHP 驱动连接：https://learn.microsoft.com/en-us/sql/connect/php/connection-options
