```
title: MSQL服务无法启动
date: 2020-01-22 19:05:37
tags: MySQL
keywords: MySQL
categories: 实操
```



# MySQL 服务正在启动。MSQL服务无法启动。服务没有报告任何错误。请键入NET HELPMSG 3534 以获得更多的帮助。（双系统修复二）

因为我以前下过mysql，所以这次懒得在官网重新下载，因此碰到了不少的麻烦。

1.通过DOS窗口输入net start mysql时，却提示服务名无效

解决方案：

（1）首先我们先进入mysql的安装目录下的bin目录

（2）之后打开DOS命令窗口（一定要管理员身份打开，不然会报错），进入该目录下（一定要进入该目录，否则操作错误）。

（3）输入命令：mysqld --install。提示安装服务成功。

（4）如果要卸载服务，可以输入如下命令：mysqld --remove。提示移除服务成功。

2.MySQL 服务正在启动。MSQL服务无法启动。服务没有报告任何错误。请键入NET HELPMSG 3534 已获得更多的帮助。

解决方案：

> mysqld  --initialize 初始化data目录即可。