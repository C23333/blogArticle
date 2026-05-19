---
title: SQL Server 1433连不上
date: 2022-04-07 22:06:00
tags:
  - SQL Server
  - CentOS
  - 端口
keywords: SQL Server 1433,mssql-conf,firewalld,sqlcmd,CentOS7
categories: 实操
---



# SQL Server 1433连不上

SQL Server 装完以后，本机 `sqlcmd` 能进，但是应用机连 `10.10.20.11,1433` 一直失败。

这种问题不要一开始就怀疑账号密码。先确认 1433 到底有没有监听、有没有放行。

## 先看服务

```bash
systemctl status mssql-server
```

查版本：

```bash
/opt/mssql-tools/bin/sqlcmd -S 127.0.0.1 -U sa -P '******' -Q "SELECT @@VERSION"
```

本机能进，说明 SQL Server 服务基本正常。

## 看 1433 是否监听

```bash
ss -lntp | grep 1433
```

正常应该能看到类似：

```text
LISTEN 0 128 *:1433 *:* users:(("sqlservr",pid=1234,fd=123))
```

如果没监听，先看 SQL Server TCP 配置。

```bash
/opt/mssql/bin/mssql-conf get network.tcpport
```

设置端口：

```bash
/opt/mssql/bin/mssql-conf set network.tcpport 1433
systemctl restart mssql-server
```

再查：

```bash
ss -lntp | grep 1433
```

## 看防火墙

CentOS 7 上 firewalld 经常忘记开端口。

```bash
firewall-cmd --state
firewall-cmd --list-all
```

放行：

```bash
firewall-cmd --permanent --add-port=1433/tcp
firewall-cmd --reload
firewall-cmd --list-ports
```

## 从应用机测端口

应用机上：

```bash
nc -vz 10.10.20.11 1433
# 或
telnet 10.10.20.11 1433
```

如果端口都不通，还是网络或防火墙。

如果端口通，但 `sqlcmd` 不通，再看账号、密码、库名、加密配置。

```bash
sqlcmd -S 10.10.20.11,1433 -U zentao_app -P '******' -d zentao -Q "SELECT 1"
```

## 看 SQL Server 错误日志

```bash
journalctl -u mssql-server -n 100 --no-pager
```

SQL Server 自己的 errorlog 也可以看：

```bash
ls -lh /var/opt/mssql/log/
tail -n 100 /var/opt/mssql/log/errorlog
```

## 常见几种情况

### 1. 本机能连，远程不能连

大概率是 1433 没监听到外部，或者防火墙没放。

### 2. 端口通，但登录失败

看登录名、密码、默认数据库、用户映射。

```sql
SELECT name, is_disabled
FROM sys.sql_logins
WHERE name = 'zentao_app';
```

### 3. 登录成功，但提示不能打开数据库

login 有了，但数据库 user 没建，或者默认数据库不可用。

```sql
USE zentao;
GO
SELECT name FROM sys.database_principals WHERE name = 'zentao_app';
GO
```

### 4. 客户安全组还有一层

VPS 控制台安全组、防火墙、系统 firewalld 可能都有规则。系统里放了端口，不代表云平台层面也放了。

## 最后记一下

1433 连不上时，顺序最好固定：

```text
服务状态 -> 本机sqlcmd -> 端口监听 -> 系统防火墙 -> 云安全组 -> 应用机telnet/nc -> 账号权限
```

别上来就改连接串。

## 参考

* SQL Server on Linux 配置：https://learn.microsoft.com/en-us/sql/linux/sql-server-linux-configure-mssql-conf
* SQL Server Red Hat 安装和连接：https://learn.microsoft.com/en-us/sql/linux/quickstart-install-connect-red-hat
