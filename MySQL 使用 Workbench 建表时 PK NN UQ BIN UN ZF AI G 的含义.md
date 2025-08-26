---
title: MySQL 使用 Workbench 建表时 PK NN UQ BIN UN ZF AI G 的含义
date: 2020-01-22 19:57:21
tags: MySQL
keywords: MySQL
categories: 理论
---



# MySQL 使用 Workbench 建表时 PK NN UQ BIN UN ZF AI G 的含义

PK - Belongs to primary key
作为主键

**主键（primary key） 一列（或一组列），其值能够唯一区分表中的每个行。
唯一标识表中每行的这个列（或这组列）称为主键。没有主键，更新或删除表中特定行很困难，因为没有安全的方法保证只设计相关的行。**



NN - Not Null
非空

UQ - Unique index
不能重复

BIN - Is binary column
存放二进制数据的列

UN - Unsigned data type
无符号数据类型（例如-500 to 500替换成0 - 1000,需要整数形数据）

ZF - Fill up values for that column with 0’s if it is numeric
填充0位（例如指定3位小数，整数18就会变成18.000）

AI - Auto Incremental
自增长

G - Generated column
基于其他列的公式生成值的列

