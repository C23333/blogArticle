---
title: MySQL 8.0.x 安装后配置远程链接和修改root密码
date: 2024-01-03 17:02:58
tags: 常用技术
keywords: MySQL
categories: 实操
top_img: https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202401031703839.png
---



# MySQL 8.0.x 安装后配置远程链接和修改root密码



### 1.安装MySQL

~~~bash
sudo apt install mysql-server
~~~



### 2.登录MySQL，修改root密码

~~~bash
# 登录MySQL
sudo mysql

# 换到MySQL数据库
use mysql;

# 修改密码、刷新权限
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '410000';
FLUSH PRIVILEGES;
~~~



### 3.配置远程连接用户

~~~bash
CREATE USER 'root'@'%' IDENTIFIED BY '410000';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
~~~

修改配置文件中MySQL的监听ip和端口

~~~bash
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
~~~

找到 `bind-address = 127.0.0.1` 并将其更改为 `#bind-address = 127.0.0.1`



### 4.重启MySQL

~~~
sudo systemctl restart mysql
~~~

