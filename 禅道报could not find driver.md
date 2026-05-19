---
title: 禅道报could not find driver
date: 2022-04-06 21:40:00
tags:
  - 禅道
  - PHP
  - SQL Server
keywords: could not find driver,pdo_sqlsrv,sqlsrv,PHP7.4,SQL Server
categories: 实操
---



# 禅道报could not find driver

禅道配置改完以后，页面直接报：

```text
could not find driver
```

这个错误看起来像数据库连不上，但其实还没到 SQL Server。PHP 这边连 SQL Server 的驱动都没有，所以 PDO 找不到 driver。

MySQL 常见是 `pdo_mysql`，SQL Server 要用 `pdo_sqlsrv`，另外还有 `sqlsrv` 扩展。

## 先看 PHP 版本

```bash
php -v
```

当时是 PHP 7.4。

再看扩展：

```bash
php -m | grep -E 'PDO|sqlsrv|pdo_sqlsrv'
```

如果只有：

```text
PDO
```

没有 `pdo_sqlsrv`，那就对上了。

## 装 Microsoft ODBC Driver

CentOS 7 上先加微软源。

```bash
curl https://packages.microsoft.com/config/rhel/7/prod.repo > /etc/yum.repos.d/mssql-release.repo
ACCEPT_EULA=Y yum install -y msodbcsql17 mssql-tools unixODBC-devel
```

把工具加到 PATH，方便用 `sqlcmd`。

```bash
echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bash_profile
source ~/.bash_profile
```

测试：

```bash
sqlcmd -? | head
```

## 安装 PHP 扩展

先装编译依赖：

```bash
yum install -y php-devel php-pear gcc gcc-c++ make unixODBC-devel
```

再装扩展：

```bash
pecl install sqlsrv
pecl install pdo_sqlsrv
```

如果 pecl 下载慢，先确认客户 VPS 能不能访问外网。有些客户环境出网受限，这个要提前准备离线包。

## 启用扩展

```bash
cat > /etc/php.d/30-sqlsrv.ini <<'EOF'
extension=sqlsrv.so
EOF

cat > /etc/php.d/35-pdo_sqlsrv.ini <<'EOF'
extension=pdo_sqlsrv.so
EOF
```

重启：

```bash
systemctl restart php-fpm
```

检查 CLI：

```bash
php -m | grep -E 'sqlsrv|pdo_sqlsrv'
```

检查 FPM 也要看。很多时候 CLI 有扩展，页面还是报错，是因为 FPM 没重启或者加载的 ini 不一样。

```bash
php --ini
systemctl status php-fpm
```

也可以临时放一个页面看：

```bash
cat > /data/www/zentao/www/sqlsrv_check.php <<'EOF'
<?php
var_dump(extension_loaded('sqlsrv'));
var_dump(extension_loaded('pdo_sqlsrv'));
EOF
```

访问后如果两个都是 `true`，驱动层基本就过了。

## 用 PHP 测一下连接

```bash
cat > /data/www/zentao/www/sqlsrv_test.php <<'EOF'
<?php
$dsn = "sqlsrv:Server=10.10.20.11,1433;Database=zentao;TrustServerCertificate=true";
$user = "zentao_app";
$pass = "<DB_PASSWORD>";

try {
    $pdo = new PDO($dsn, $user, $pass);
    $row = $pdo->query("SELECT DB_NAME() AS dbname")->fetch(PDO::FETCH_ASSOC);
    var_dump($row);
} catch (Throwable $e) {
    echo $e->getMessage();
}
EOF
```

访问：

```bash
curl http://127.0.0.1/sqlsrv_test.php
```

能输出 `zentao`，再去看禅道配置。

## 这里容易搞混

1. `sqlcmd` 能连，只能说明系统 ODBC 和网络大概率没问题。
2. `php -m` 有扩展，只能说明 CLI 加载了。
3. 页面还要看 PHP-FPM 是否加载扩展。
4. `could not find driver` 基本不是账号密码问题，是 PHP PDO 驱动没加载。

## 参考

* Microsoft PHP Drivers for SQL Server：https://learn.microsoft.com/en-us/sql/connect/php/microsoft-php-driver-for-sql-server
* Linux/macOS 安装 PHP SQL Server 驱动：https://learn.microsoft.com/en-us/sql/connect/php/installation-tutorial-linux-mac
