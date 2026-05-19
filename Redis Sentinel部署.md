---
title: Redis Sentinel部署
date: 2022-04-15 21:58:00
tags:
  - Redis
  - Sentinel
  - 禅道
keywords: Redis Sentinel,哨兵,failover,禅道,Redis6
categories: 实操
---



# Redis Sentinel部署

禅道应用做多节点以后，session 不能只放本机文件里。客户这边给了三台 Redis 节点，要求用 Sentinel 做主从切换。

这里先记 Redis Sentinel 的基础部署。IP 都脱敏。

```text
redis01 10.10.30.11
redis02 10.10.30.12
redis03 10.10.30.13
```

## 安装 Redis

CentOS 7 上可以用 yum，也可以源码编译。客户环境为了统一版本，当时走的是 Redis 6.x 源码编译。

```bash
yum install -y gcc make tcl wget
cd /usr/local/src
wget https://download.redis.io/releases/redis-6.2.6.tar.gz
tar zxvf redis-6.2.6.tar.gz
cd redis-6.2.6
make
make install
```

创建目录：

```bash
mkdir -p /etc/redis /data/redis /var/log/redis
```

## redis.conf

redis01 先作为 master：

```ini
bind 0.0.0.0
port 6379
daemonize yes
protected-mode yes
requirepass <REDIS_PASSWORD>
masterauth <REDIS_PASSWORD>
pidfile /var/run/redis_6379.pid
logfile /var/log/redis/redis.log
dir /data/redis
appendonly yes
```

redis02/redis03 作为 replica，多一行：

```ini
replicaof 10.10.30.11 6379
```

启动：

```bash
redis-server /etc/redis/redis.conf
```

检查：

```bash
redis-cli -a '<REDIS_PASSWORD>' INFO replication
```

master 上应该看到 connected slaves。

## sentinel.conf

三台都配 Sentinel。端口 26379。

```ini
port 26379
daemonize yes
protected-mode yes
bind 0.0.0.0
logfile /var/log/redis/sentinel.log
pidfile /var/run/redis-sentinel.pid

dir /data/redis

sentinel monitor zentao-master 10.10.30.11 6379 2
sentinel auth-pass zentao-master <REDIS_PASSWORD>
sentinel down-after-milliseconds zentao-master 5000
sentinel failover-timeout zentao-master 60000
sentinel parallel-syncs zentao-master 1
```

`2` 是 quorum。三台 Sentinel 里至少两个认为 master 不可用，才会进入判断。

启动：

```bash
redis-sentinel /etc/redis/sentinel.conf
```

## 防火墙

```bash
firewall-cmd --permanent --add-port=6379/tcp
firewall-cmd --permanent --add-port=26379/tcp
firewall-cmd --reload
```

应用机测：

```bash
nc -vz 10.10.30.11 6379
nc -vz 10.10.30.11 26379
```

## 看 Sentinel 状态

```bash
redis-cli -p 26379 SENTINEL masters
redis-cli -p 26379 SENTINEL slaves zentao-master
redis-cli -p 26379 SENTINEL get-master-addr-by-name zentao-master
```

如果 Sentinel 自己也配置了 `requirepass`，再带 `-a`：

```bash
redis-cli -p 26379 -a '<REDIS_PASSWORD>' SENTINEL get-master-addr-by-name zentao-master
```

注意这个密码不是 `sentinel auth-pass` 的概念。`sentinel auth-pass` 是 Sentinel 去连 Redis master/replica 用的；`redis-cli -p 26379 -a` 是客户端连 Sentinel 本身用的。这个当时就差点搞混。

## 切主测试

停 master：

```bash
redis-cli -h 10.10.30.11 -a '<REDIS_PASSWORD>' SHUTDOWN
```

然后看 Sentinel：

```bash
redis-cli -h 10.10.30.12 -p 26379 SENTINEL get-master-addr-by-name zentao-master
```

应该变成 redis02 或 redis03。

再看新 master：

```bash
redis-cli -h 10.10.30.12 -a '<REDIS_PASSWORD>' INFO replication
```

## 当时容易错的地方

1. replica 配了 `requirepass`，但忘了 `masterauth`。
2. Sentinel 端口 26379 没放。
3. 三台机器时间不一致，日志看起来很乱。
4. 只启动 Redis，忘了启动 Sentinel。
5. 应用只写死 Redis master IP，不会问 Sentinel。
6. failover 以后旧 master 回来，要看它是否变成 replica。

## 参考

* Redis Sentinel 官方文档：https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/
* Redis replication：https://redis.io/docs/latest/operate/oss_and_stack/management/replication/
