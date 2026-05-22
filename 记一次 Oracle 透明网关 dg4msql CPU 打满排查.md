---
title: 记一次 Oracle 透明网关 dg4msql CPU 打满排查
date: 2026-05-22 15:40:00
tags:
  - Oracle
  - 透明网关
  - dg4msql
  - SQL Server
  - DBA
keywords: Oracle透明网关,dg4msql,HS message to agent,CLOSE_WAIT,DBLink,SQL Server
categories: 实操
---



# 记一次 Oracle 透明网关 dg4msql CPU 打满排查

> 说明：本文中的 IP、主机名、数据库用户、客户端用户名、客户端机器名、DBLink、表名、业务字段和值都已做脱敏处理。
> 文中的命令、排查顺序和现象判断保留原始逻辑，示例名称只用于说明问题。

2026-05-22 下午，收到一台 Oracle 透明网关服务器 CPU 打满的问题。

这台机器是 `gateway01`，IP 是 `10.10.10.10`。它上面跑的是 Oracle Database Gateway for SQL Server，进程名表现为 `dg4msqlxxx`，例如：

```text
dg4msqlgw_alpha
dg4msqlgw_beta
dg4msqlgw_gamma
dg4msqlgw_delta
```

一开始看到 `top` 的时候，机器基本已经被打满：

```text
top - 13:25:35 up 434 days, 20:44,  2 users,  load average: 110.83, 110.50, 110.41
Tasks: 367 total, 109 running, 258 sleeping,   0 stopped,   0 zombie
Cpu(s): 98.9%us,  1.1%sy,  0.0%ni,  0.0%id,  0.0%wa
```

下面全是 `dg4msql`：

```text
PID     USER    %CPU  COMMAND
16195   oracle  4.0   dg4msql
19318   oracle  4.0   dg4msql
37217   oracle  4.0   dg4msql
54834   oracle  4.0   dg4msql
...
```

这个时候不要先重启，也不要直接 `killall dg4msql`。透明网关后面可能挂着多个业务链路，先要知道到底是谁打过来的。

## 先按网关 SID 分组

先看总共有多少 `dg4msql`：

```bash
pgrep dg4msql | wc -l
```

输出：

```text
135
```

再按进程名分组，看哪个网关 SID 是大头：

```bash
ps -eo pid,pcpu,args | awk '/[d]g4msql/ {cnt[$3]++; cpu[$3]+=$2} END {for(k in cnt) printf "%8.1f %5d %s\n", cpu[k], cnt[k], k}' | sort -nr
```

输出：

```text
   125.6    85 dg4msqlgw_alpha
    29.4    17 dg4msqlgw_beta
    18.4    21 dg4msqlgw_gamma
     0.0    12 dg4msqlgw_delta
```

第一眼就很明显，`gw_alpha` 是最大头。

另外再看这些进程启动时间，会发现很多不是刚启动的：

```bash
ps -eo pid,ppid,lstart,etime,pcpu,pmem,args --sort=-pcpu | grep '[d]g4msql' | head -50
```

部分输出：

```text
188238 1 Fri Mar 14 09:21:24 2025 434-04:09:58 4.7 0.3 dg4msqlgw_beta (LOCAL=NO)
144783 1 Mon Mar 17 13:21:34 2025 431-00:09:48 3.7 0.3 dg4msqlgw_alpha (LOCAL=NO)
186502 1 Mon Mar 17 11:11:08 2025 431-02:20:14 3.7 0.3 dg4msqlgw_alpha (LOCAL=NO)
256613 1 Wed Apr  2 15:51:53 2025 414-21:39:29 2.5 0.3 dg4msqlgw_beta (LOCAL=NO)
129224 1 Thu Apr 10 14:04:42 2025 406-23:26:40 2.2 0.3 dg4msqlgw_alpha (LOCAL=NO)
```

这时候已经可以排除“刚刚突然有人开了很多连接”这种简单判断。很多进程已经跑了几百天，更像是长期会话泄漏或透明网关 agent 残留。

## 查单个 dg4msql 的 TCP 对端

挑几个高 CPU PID 看 `lsof`：

```bash
for p in 188238 144783 186502 256613 129224 99378 137385 180930; do
  echo "===== PID $p ====="
  ps -o pid,lstart,etime,pcpu,pmem,args -p $p
  lsof -Pan -p $p -iTCP 2>/dev/null
done
```

部分输出：

```text
===== PID 188238 =====
COMMAND    PID   USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
dg4msql 188238 oracle    8u  IPv4 11900735      0t0  TCP 10.10.10.10:53549->10.10.30.10:1433 (ESTABLISHED)

===== PID 129224 =====
COMMAND    PID   USER   FD   TYPE    DEVICE SIZE/OFF NODE NAME
dg4msql 129224 oracle   13u  IPv4 306489800      0t0  TCP 10.10.10.10:1521->10.10.20.10:40448 (ESTABLISHED)

===== PID 99378 =====
COMMAND   PID   USER   FD   TYPE    DEVICE SIZE/OFF NODE NAME
dg4msql 99378 oracle   13u  IPv4 315507305      0t0  TCP 10.10.10.10:1521->10.10.20.10:43722 (ESTABLISHED)
```

这里要分清两种连接：

```text
gateway01:1521 -> 来源 Oracle/应用服务器
gateway01:随机端口 -> SQL Server:1433
```

比如：

```text
10.10.10.10:1521 -> 10.10.20.10:40448
```

说明 `10.10.20.10` 是访问透明网关的来源机器。

## 反查 10.10.20.10 是哪台机器

先查 DNS，没有结果：

```bash
getent hosts 10.10.20.10
nslookup 10.10.20.10
```

输出：

```text
** server can't find 10.20.10.10.in-addr.arpa.: NXDOMAIN
```

再查 ARP：

```bash
ip neigh show 10.10.20.10
arp -an | grep '10.10.20.10'
cat /proc/net/arp | grep '10.10.20.10'
```

输出：

```text
10.10.20.10 dev eth0 lladdr 02:00:00:aa:bb:cc STALE
? (10.10.20.10) at 02:00:00:aa:bb:cc [ether] on eth0
10.10.20.10    0x1         0x2         02:00:00:aa:bb:cc     *        eth0
```

再查 listener 日志。`root` 下没有 `lsnrctl`，切到 `oracle` 用户：

```bash
su - oracle
lsnrctl status | grep -i "Listener Log File"
```

输出：

```text
Listener Log File <ORACLE_BASE>/diag/tnslsnr/gateway01/listener/alert/log.xml
```

查 `10.10.20.10`：

```bash
grep '10.10.20.10' "<ORACLE_BASE>/diag/tnslsnr/gateway01/listener/alert/log.xml" | tail -100
```

里面全是这种：

```text
<txt>18-MAY-2026 10:01:10 * (CONNECT_DATA=(SID=gw_alpha)(CID=(PROGRAM=)(HOST=source-host.local)(USER=os_admin))) * (ADDRESS=(PROTOCOL=tcp)(HOST=10.10.20.10)(PORT=44946)) * establish * gw_alpha * 0
<txt>18-MAY-2026 11:01:10 * (CONNECT_DATA=(SID=gw_alpha)(CID=(PROGRAM=)(HOST=source-host.local)(USER=os_admin))) * (ADDRESS=(PROTOCOL=tcp)(HOST=10.10.20.10)(PORT=46702)) * establish * gw_alpha * 0
<txt>18-MAY-2026 12:01:10 * (CONNECT_DATA=(SID=gw_alpha)(CID=(PROGRAM=)(HOST=source-host.local)(USER=os_admin))) * (ADDRESS=(PROTOCOL=tcp)(HOST=10.10.20.10)(PORT=48438)) * establish * gw_alpha * 0
```

每小时一次，`USER=os_admin`，`HOST=source-host.local`。

一开始我还以为是某台 Laravel 定时任务机器，但后面登录确认它其实是 某 ERP 系统 的 Oracle 服务器。

## 登录 10.10.20.10，确认是 源库A服务器

登录以后先看 IP：

```bash
ip a
```

输出：

```text
2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    link/ether 02:00:00:aa:bb:cc
    inet 10.10.20.10/24 brd 10.10.20.255 scope global ens18
```

对上了。

再查它到透明网关的连接：

```bash
ss -antp | grep '10.10.10.10:1521'
```

输出很多：

```text
ESTAB 0 0 10.10.20.10:55388 10.10.10.10:1521 users:(("oracle_220743_x",pid=220743,fd=12))
ESTAB 0 0 10.10.20.10:44526 10.10.10.10:1521 users:(("oracle_155203_x",pid=155203,fd=20))
ESTAB 0 0 10.10.20.10:20150 10.10.10.10:1521 users:(("oracle_78115_x3",pid=78115,fd=12))
...
```

再看进程：

```bash
ps -eo pid,ppid,lstart,etime,pcpu,pmem,args --sort=start_time | grep -Ei 'oracle|sqlplus|adx|sad|syracuse|x3|node|java' | grep -v grep
```

能看到大量 `oracleAPPDB1 (LOCAL=NO)`，很多也是 2025 年开始的：

```text
154528 1 Thu Apr 10 14:04:25 2025 406-23:48:57 oracleAPPDB1 (LOCAL=NO)
164293 1 Thu Apr 10 14:09:43 2025 406-23:43:39 oracleAPPDB1 (LOCAL=NO)
212135 1 Thu Apr 10 14:36:21 2025 406-23:17:01 oracleAPPDB1 (LOCAL=NO)
...
```

这就说明 源库A里确实有会话长期持有透明网关连接。

## 回 源库A查对应 session

先挑一个 PID，例如 `238721`：

```bash
ps -fp 238721
lsof -Pan -p 238721 -iTCP 2>/dev/null
```

输出：

```text
UID         PID   PPID  C STIME TTY          TIME CMD
oracle   238721      1  0  2025 ?        00:00:00 oracleAPPDB1 (LOCAL=NO)

COMMAND      PID   USER   FD   TYPE     DEVICE SIZE/OFF NODE NAME
oracle_23 238721 oracle   12u  IPv4 2073001967      0t0  TCP 10.10.20.10:55694->10.10.10.10:1521 (ESTABLISHED)
```

进 Oracle 查 `v$session`：

```sql
select p.spid,
       s.sid,
       s.serial#,
       s.username,
       s.osuser,
       s.machine,
       s.program,
       s.module,
       s.action,
       s.client_identifier,
       s.status,
       s.event,
       s.sql_id,
       s.prev_sql_id,
       to_char(s.logon_time, 'yyyy-mm-dd hh24:mi:ss') logon_time,
       s.last_call_et
from v$process p
join v$session s on s.paddr = p.addr
where p.spid = '238721';
```

输出：

```text
SPID    SID  SERIAL# USERNAME OSUSER MACHINE                    PROGRAM       MODULE            ACTION
238721  197  34177   APP_USER_A      user_a    DOMAIN\DEV-CLIENT-01  plsqldev.exe PL/SQL Developer app_debug.sql

STATUS EVENT               PREV_SQL_ID   LOGON_TIME           LAST_CALL_ET
ACTIVE HS message to agent sqlid_txn 2025-05-30 15:47:18  30837490
```

这个就非常明确了：不是 ERP 服务本身，是一个 PL/SQL Developer 人工客户端会话，从 2025-05-30 一直挂到现在。

## 批量查 源库A上的会话

把几个 OS PID 一起查：

```sql
select p.spid,
       s.sid,
       s.serial#,
       s.username,
       s.osuser,
       s.machine,
       s.program,
       s.module,
       s.action,
       s.status,
       s.event,
       s.sql_id,
       s.prev_sql_id,
       to_char(s.logon_time, 'yyyy-mm-dd hh24:mi:ss') logon_time,
       s.last_call_et
from v$process p
join v$session s on s.paddr = p.addr
where p.spid in (
'164293','168234','170855','197864','198353',
'212135','214143','220743','228123','231690','238721'
)
order by s.logon_time;
```

输出摘取：

```text
SPID    SID SERIAL# USERNAME OSUSER MACHINE                     PROGRAM      ACTION                              STATUS EVENT
164293  425 37127   APP_USER_A      user_d  DOMAIN\DEV-CLIENT-02   plsqldev.exe SQL Window                          ACTIVE HS message to agent
212135   67 26694   APP_USER_A      user_d  DOMAIN\DEV-CLIENT-02   plsqldev.exe SELECT * FROM LOCAL_TABLE_X FOR ...      ACTIVE HS message to agent
168234  371 65448   APP_USER_A      user_e  DOMAIN\DEV-CLIENT-03   plsqldev.exe SQL ??                              ACTIVE HS message to agent
220743  437 38771   APP_USER_A      user_a    DOMAIN\DEV-CLIENT-01   plsqldev.exe app_debug.sql                            ACTIVE HS message to agent
238721  197 34177   APP_USER_A      user_a    DOMAIN\DEV-CLIENT-01   plsqldev.exe app_debug.sql                            ACTIVE HS message to agent
198353   14 65532   APP_USER_A      user_a    DOMAIN\DEV-CLIENT-04   plsqldev.exe app_query.sql                           ACTIVE HS message to agent
228123  319     0   APP_USER_A      user_a    DEV-CLIENT-05             DataGrip                                         ACTIVE HS message to agent
```

再看 SQL：

```sql
select s.sid,
       s.serial#,
       p.spid,
       nvl(s.sql_id, s.prev_sql_id) sql_id,
       dbms_lob.substr(q.sql_fulltext, 3000, 1) sql_text
from v$process p
join v$session s on s.paddr = p.addr
left join v$sqlarea q on q.sql_id = nvl(s.sql_id, s.prev_sql_id)
where p.spid in (
'164293','168234','170855','197864','198353',
'212135','214143','220743','228123','231690','238721'
)
order by s.sid;
```

输出：

```text
SID SERIAL# SPID   SQL_ID        SQL_TEXT
14  65532   198353 sqlid_e SELECT * FROM REMOTE_TABLE_A@dblink_alpha
67  26694   212135 sqlid_b SELECT * FROM remote_table_b@dblink_alpha
88   6497   197864 sqlid_f SELECT * FROM REMOTE_TABLE_A@DBLINK_GAMMA
197 34177   238721 sqlid_txn begin :id := sys.dbms_transaction.local_transaction_id; end;
242 22952   231690 sqlid_c SELECT * FROM REMOTE_TABLE_B@DBLINK_ALPHA
366 26154   170855 sqlid_c SELECT * FROM REMOTE_TABLE_B@DBLINK_ALPHA
371 65448   168234 sqlid_c SELECT * FROM REMOTE_TABLE_B@DBLINK_ALPHA
372 42836   214143 sqlid_d SELECT * FROM REMOTE_TABLE_A@DBLINK_ALPHA WHERE CREATOR='XX'
425 37127   164293 sqlid_a SELECT * FROM remote_table_c@dblink_alpha
437 38771   220743 sqlid_txn begin :id := sys.dbms_transaction.local_transaction_id; end;
```

这批基本可以定性为人工客户端远程表查询，很多是 `select * from xxx@dblink`，并且已经卡了几百天。

## 从 Oracle 侧 kill session

先从源库断 session：

```sql
alter system kill session '425,37127' immediate;
alter system kill session '67,26694' immediate;
alter system kill session '371,65448' immediate;
alter system kill session '366,26154' immediate;
alter system kill session '372,42836' immediate;
alter system kill session '437,38771' immediate;
alter system kill session '197,34177' immediate;
alter system kill session '14,65532' immediate;
alter system kill session '242,22952' immediate;
alter system kill session '88,6497' immediate;
alter system kill session '319,0' immediate;
```

再查残留：

```sql
select sid, serial#, username, machine, program, module, status, event, logon_time, last_call_et
from v$session
where type = 'USER'
  and event = 'HS message to agent'
order by logon_time;
```

当时还剩 3 个：

```text
SID=87,  SERIAL#=0      APP_USER_A / plsqldev.exe / DOMAIN\DEV-CLIENT-03 / 2025-07-03
SID=6,   SERIAL#=25460  APP_USER_A / plsqldev.exe / DOMAIN\DEV-CLIENT-04 / 2025-07-23
SID=319, SERIAL#=0      APP_USER_A / DataGrip     / DEV-CLIENT-05           / 2025-12-25
```

这类 `SERIAL#=0` 有时不太好 kill，必要时用 `disconnect session` 或 OS 层处理。

## 为什么源库 session 清了，gateway01 上 dg4msql 还在

回 `gateway01` 看，`dg4msqlgw_alpha` 数量没有立刻掉下来。

继续查连接状态：

```bash
for p in $(pgrep -f 'dg4msqlgw_alpha'); do
  if lsof -Pan -p $p -iTCP 2>/dev/null | grep -q '10.10.20.10'; then
    echo "===== $p ====="
    ps -o pid,lstart,etime,pcpu,args -p $p
    lsof -Pan -p $p -iTCP 2>/dev/null
  fi
done
```

输出里大量是 `CLOSE_WAIT`：

```text
===== 14919 =====
PID     STARTED                  ELAPSED       %CPU COMMAND
14919   Tue Jul 29 14:38:16 2025 296-23:34:27 1.6  dg4msqlgw_alpha (LOCAL=NO)
dg4msql 14919 oracle 13u TCP 10.10.10.10:1521->10.10.20.10:28096 (CLOSE_WAIT)

===== 16195 =====
dg4msql 16195 oracle 13u TCP 10.10.10.10:1521->10.10.20.10:59876 (CLOSE_WAIT)
```

`CLOSE_WAIT` 的意思是：对端已经关了，`gateway01` 这边进程没有退出。

先温和 kill 没用：

```bash
kill PID
```

很多 `dg4msql` 不响应，最后只能限定范围后 `kill -9`。

清 源库A来源的 `CLOSE_WAIT`：

```bash
for p in $(pgrep -f 'dg4msqlgw_alpha'); do
  if lsof -Pan -p $p -iTCP 2>/dev/null | grep '10.10.20.10' | grep -q 'CLOSE_WAIT'; then
    echo $p
  fi
done > /tmp/kill_gw_alpha_close_wait.pids

xargs -r kill -9 < /tmp/kill_gw_alpha_close_wait.pids
```

清完以后：

```text
gw_alpha 85 -> 42
```

CPU 明显下降，但还没完全恢复。

## 再查 NO_TCP 和 NO_CLIENT_TO_SQLSERVER

剩余有一批进程，已经看不到来源 IP：

```bash
for p in $(ps -eo pid,pcpu,args --sort=-pcpu | awk '/[d]g4msql/ {print $1}' | head -80); do
  sid=$(ps -o args= -p "$p" | awk '{print $1}')
  cpu=$(ps -o pcpu= -p "$p" | awk '{print $1}')
  tcp=$(lsof -Pan -p "$p" -iTCP 2>/dev/null | awk 'NR>1 {print}')
  if [ -z "$tcp" ]; then
    echo "NO_TCP $sid $cpu $p"
  else
    echo "$tcp" | awk -v sid="$sid" -v cpu="$cpu" -v pid="$p" '
      /TCP/ {
        state=$NF
        print state, sid, cpu, pid
      }
    '
  fi
done | awk '
{
  key=$1" "$2
  cnt[key]++
  cpu[key]+=$3
}
END {
  for (k in cnt) printf "%8.1f %5d %s\n", cpu[k], cnt[k], k
}
' | sort -nr
```

输出：

```text
    41.9    25 NO_TCP dg4msqlgw_alpha
    32.1    19 (ESTABLISHED) dg4msqlgw_beta
    17.1    14 (ESTABLISHED) dg4msqlgw_gamma
    12.6    19 (ESTABLISHED) dg4msqlgw_alpha
     6.1     4 (CLOSE_WAIT) dg4msqlgw_gamma
     4.8     3 NO_TCP dg4msqlgw_beta
     2.7     2 NO_TCP dg4msqlgw_gamma
```

`NO_TCP` 表示进程已经没有任何 TCP 连接，但还在吃 CPU。这个基本就是孤儿进程。

清 `NO_TCP dg4msqlgw_alpha`：

```bash
for p in $(pgrep -f 'dg4msqlgw_alpha'); do
  if ! lsof -Pan -p "$p" -iTCP >/dev/null 2>&1; then
    echo $p
  fi
done | sort -n | uniq > /tmp/kill_gw_alpha_no_tcp_orphans.pids

xargs -r kill -9 < /tmp/kill_gw_alpha_no_tcp_orphans.pids
```

再处理没有客户端、只剩 SQL Server 连接的进程：

```bash
for p in $(pgrep -f 'dg4msql'); do
  sid=$(ps -o args= -p "$p" | awk '{print $1}')
  tcp=$(lsof -Pan -p "$p" -iTCP 2>/dev/null)

  has_client=$(echo "$tcp" | grep ':1521->')
  has_sqlserver=$(echo "$tcp" | grep ':1433')
  has_closewait=$(echo "$tcp" | grep 'CLOSE_WAIT')

  if [ -n "$has_closewait" ]; then
    echo "$p $sid CLOSE_WAIT"
  elif [ -z "$has_client" ] && [ -n "$has_sqlserver" ]; then
    echo "$p $sid NO_CLIENT_TO_SQLSERVER"
  fi
done | sort -n > /tmp/kill_gateway_orphans.pids.txt
```

当时生成 26 个：

```text
5497 dg4msqlgw_gamma CLOSE_WAIT
22227 dg4msqlgw_gamma CLOSE_WAIT
37217 dg4msqlgw_alpha NO_CLIENT_TO_SQLSERVER
54834 dg4msqlgw_gamma NO_CLIENT_TO_SQLSERVER
55657 dg4msqlgw_beta NO_CLIENT_TO_SQLSERVER
56480 dg4msqlgw_beta NO_CLIENT_TO_SQLSERVER
68232 dg4msqlgw_alpha NO_CLIENT_TO_SQLSERVER
...
256613 dg4msqlgw_beta NO_CLIENT_TO_SQLSERVER
```

这批大多也是几十天到几百天的进程，清理：

```bash
awk '{print $1}' /tmp/kill_gateway_orphans.pids.txt | xargs -r kill -9
```

清完以后，负载从一开始的 110 左右降到 35 左右，但还有少数长连接继续打 CPU。

## 按来源 IP 分类剩余连接

继续按来源 IP 分类：

```bash
for p in $(pgrep -f 'dg4msql'); do
  sid=$(ps -o args= -p "$p" | awk '{print $1}')
  cpu=$(ps -o pcpu= -p "$p" | awk '{print $1}')
  etime=$(ps -o etime= -p "$p" | awk '{print $1}')
  tcp=$(lsof -Pan -p "$p" -iTCP 2>/dev/null)

  src=$(echo "$tcp" | awk '/:1521->/ {peer=$9; sub(/^.*->/,"",peer); split(peer,a,":"); print a[1]; exit}')
  dst=$(echo "$tcp" | awk '/:1433/ {peer=$9; sub(/^.*->/,"",peer); split(peer,a,":"); print a[1]; exit}')
  state=$(echo "$tcp" | awk '/:1521->/ {print $NF; exit}')

  [ -z "$src" ] && src="NO_CLIENT"
  [ -z "$dst" ] && dst="NO_SQLSERVER"
  [ -z "$state" ] && state="NO_CLIENT_STATE"

  echo "$src $sid $dst $state $cpu $etime $p"
done | awk '
{
  key=$1" "$2" -> "$3" "$4
  cnt[key]++
  cpu[key]+=$5
}
END {
  for (k in cnt) printf "%8.1f %5d %s\n", cpu[k], cnt[k], k
}
' | sort -nr
```

输出：

```text
     4.4     3 10.10.20.21 dg4msqlgw_beta -> 10.10.30.10 (ESTABLISHED)
     3.0     2 10.10.20.22 dg4msqlgw_beta -> 10.10.30.10 (ESTABLISHED)
     1.5     1 10.10.20.13 dg4msqlgw_alpha -> NO_SQLSERVER (ESTABLISHED)
     1.4     1 10.10.20.22 dg4msqlgw_gamma -> 10.10.30.11 (ESTABLISHED)
     1.4     1 10.10.20.23 dg4msqlgw_alpha -> NO_SQLSERVER (ESTABLISHED)
```

再输出明细：

```text
SRC_IP          SID                SQLSERVER       STATE         CPU      ELAPSED      PID
10.10.20.22  dg4msqlgw_beta   10.10.30.10    (ESTABLISHED) 1.7      8-00:01:39   122285
10.10.20.21  dg4msqlgw_beta   10.10.30.10    (ESTABLISHED) 1.7      18-03:55:48  229465
10.10.20.13     dg4msqlgw_alpha    NO_SQLSERVER    (ESTABLISHED) 1.5      36-00:05:16  157423
10.10.20.22  dg4msqlgw_gamma    10.10.30.11   (ESTABLISHED) 1.4      49-20:21:45  191946
10.10.20.21  dg4msqlgw_beta   10.10.30.10    (ESTABLISHED) 1.4      36-00:37:45  124324
10.10.20.23  dg4msqlgw_alpha    NO_SQLSERVER    (ESTABLISHED) 1.4      40-21:33:27  181095
10.10.20.22  dg4msqlgw_beta   10.10.30.10    (ESTABLISHED) 1.3      3-01:52:35   154075
10.10.20.21  dg4msqlgw_beta   10.10.30.10    (ESTABLISHED) 1.3      52-01:07:25  118860
```

这时剩余问题已经很集中：

```text
10.10.20.21
10.10.20.22
10.10.20.23
10.10.20.13
```

## 查 10.10.20.21：srcdb01

登录 `10.10.20.21`，机器名是 `srcdb01`。

```bash
ss -antp | grep '10.10.10.10:1521'
```

输出：

```text
ESTAB 0 0 10.10.20.21:55655 10.10.10.10:1521 users:(("oracle",pid=142731,fd=19))
ESTAB 0 0 10.10.20.21:43661 10.10.10.10:1521 users:(("oracle",pid=209042,fd=13))
ESTAB 0 0 10.10.20.21:14401 10.10.10.10:1521 users:(("oracle",pid=151730,fd=14))
ESTAB 0 0 10.10.20.21:43401 10.10.10.10:1521 users:(("oracle",pid=209042,fd=11))
ESTAB 0 0 10.10.20.21:55651 10.10.10.10:1521 users:(("oracle",pid=142731,fd=15))
```

看进程：

```bash
ps -fp 142731 151730 209042
```

输出：

```text
UID       PID    PPID C STIME TTY TIME     CMD
oracle    142731 1    0 Mar31 ?   0:45     oracleSRCDB2 (LOCAL=NO)
oracle    151730 1    0 May04 ?   0:54     oracleSRCDB2 (LOCAL=NO)
oracle    209042 1    0 Apr16 ?   3:42     oracleSRCDB2 (LOCAL=NO)
```

进 Oracle 查：

```sql
select p.spid,
       s.sid,
       s.serial#,
       s.username,
       s.osuser,
       s.machine,
       s.program,
       s.module,
       s.action,
       s.status,
       s.event,
       s.sql_id,
       s.prev_sql_id,
       to_char(s.logon_time, 'yyyy-mm-dd hh24:mi:ss') logon_time,
       s.last_call_et
from v$process p
join v$session s on s.paddr = p.addr
where p.spid in ('142731','151730','209042')
order by s.logon_time;
```

输出：

```text
SPID    SID  SERIAL# USERNAME OSUSER MACHINE                PROGRAM       MODULE             STATUS EVENT
142731  1252 59111   APP_USER_B       user_a    DEV-MAC-01      DataGrip      DataGrip           ACTIVE HS message to agent
209042   484 46997   APP_USER_B       user_b DOMAIN\DEV-CLIENT-06     plsqldev.exe PL/SQL Developer   ACTIVE HS message to agent
```

SQL：

```text
SPID   SQL_TEXT
209042 SELECT * FROM remote_table_e@dblink_zeta
```

断：

```sql
alter system kill session '1252,59111' immediate;
alter system kill session '484,46997' immediate;
```

`151730` 查不到 session，但 OS 进程和 TCP 还在：

```bash
ss -antp | grep '14401'
ps -fp 151730
```

输出：

```text
ESTAB 0 0 10.10.20.21:14401 10.10.10.10:1521 users:(("oracle",pid=151730,fd=14))
oracle 151730 1 0 May04 ? 00:00:54 oracleSRCDB2 (LOCAL=NO)
```

这种已经没有 `gv$session` 记录，只能清 OS 残留：

```bash
kill 151730
# 不退再 kill -9 151730
```

对应 `gateway01` 上的 `229465` 后面也清掉：

```bash
kill -9 229465
```

## 查 10.10.20.22：srcdb02

登录 `10.10.20.22`，机器名是 `srcdb02`。

```bash
ss -antp | grep '10.10.10.10:1521'
```

输出：

```text
ESTAB 0 0 10.10.20.22:36590 10.10.10.10:1521 users:(("oracle",pid=85707,fd=11))
ESTAB 0 0 10.10.20.22:21996 10.10.10.10:1521 users:(("oracle",pid=102630,fd=23))
ESTAB 0 0 10.10.20.22:33387 10.10.10.10:1521 users:(("oracle",pid=102630,fd=11))
ESTAB 0 0 10.10.20.22:14258 10.10.10.10:1521 users:(("oracle",pid=158683,fd=11))
ESTAB 0 0 10.10.20.22:27958 10.10.10.10:1521 users:(("oracle",pid=105163,fd=11))
ESTAB 0 0 10.10.20.22:52863 10.10.10.10:1521 users:(("oracle",pid=14467,fd=14))
```

查 session：

```sql
select p.inst_id,
       p.spid,
       s.sid,
       s.serial#,
       s.username,
       s.osuser,
       s.machine,
       s.program,
       s.module,
       s.action,
       s.status,
       s.event,
       s.sql_id,
       s.prev_sql_id,
       to_char(s.logon_time, 'yyyy-mm-dd hh24:mi:ss') logon_time,
       s.last_call_et
from gv$process p
join gv$session s
  on s.inst_id = p.inst_id
 and s.paddr = p.addr
where p.spid in ('85707','102630','158683','105163','14467')
order by s.logon_time;
```

输出：

```text
INST_ID SPID   SID  SERIAL# USERNAME OSUSER MACHINE                 PROGRAM      STATUS   EVENT
2       14467  127  28947   APP_USER_D   user_c   DOMAIN\DEV-CLIENT-07 plsqldev.exe ACTIVE HS message to agent
2       102630 84   18231   APP_USER_C      user_b DOMAIN\DEV-CLIENT-06       plsqldev.exe ACTIVE HS message to agent
2       85707  1290 45802   APP_USER_C      user_b DOMAIN\DEV-CLIENT-06       plsqldev.exe ACTIVE HS message to agent
2       158683 557  52409   APP_USER_C      user_b DOMAIN\DEV-CLIENT-08         plsqldev.exe INACTIVE SQL*Net message from client
2       105163 1451 29031   APP_USER_C      user_b DOMAIN\DEV-CLIENT-06       plsqldev.exe INACTIVE SQL*Net message from client
```

查 SQL：

```text
SID SERIAL# SPID   SQL_TEXT
84  18231   102630 select * from remote_table_f@dblink_zeta where "biz_key" in('KEY_001', 'KEY_002')
127 28947   14467  begin :id := sys.dbms_transaction.local_transaction_id; end;
557 52409   158683 begin :id := sys.dbms_transaction.local_transaction_id; end;
1451 29031  105163 begin :id := sys.dbms_transaction.local_transaction_id; end;
```

先断 ACTIVE 的：

```sql
alter system kill session '127,28947,@2' immediate;
alter system kill session '84,18231,@2' immediate;
alter system kill session '1290,45802,@2' immediate;
```

其中第三个当时返回：

```text
ORA-00030: User session ID does not exist.
```

这说明会话已经没了。

回 `gateway01` 查，对应 PID 已经变成 `CLOSE_WAIT`：

```bash
lsof -Pan -p 191946 -iTCP 2>/dev/null
lsof -Pan -p 122285 -iTCP 2>/dev/null
lsof -Pan -p 154075 -iTCP 2>/dev/null
```

输出：

```text
dg4msql 191946 oracle ... TCP 10.10.10.10:1521->10.10.20.22:52863 (CLOSE_WAIT)
dg4msql 122285 oracle ... TCP 10.10.10.10:1521->10.10.20.22:21996 (CLOSE_WAIT)
```

清残留：

```bash
kill -9 191946 122285 154075
```

## 查 10.10.20.23：srcdb03

最后剩下两个大头：

```text
157423 -> 10.10.20.13:45647
181095 -> 10.10.20.23:13179
```

先查 `10.10.20.23`，机器名是 `srcdb03`：

```bash
ss -antp | grep '10.10.10.10:1521' | grep ':13179'
```

输出：

```text
ESTAB 0 0 10.10.20.23:13179 10.10.10.10:1521 users:(("oracle",pid=128698,fd=14))
```

查进程：

```bash
ps -fp 128698
```

输出：

```text
oracle 128698 1 0 Apr11 ? 00:00:00 oracleSRCDB1 (LOCAL=NO)
```

查 Oracle session：

```sql
select p.inst_id,
       p.spid,
       s.sid,
       s.serial#,
       s.username,
       s.osuser,
       s.machine,
       s.program,
       s.module,
       s.action,
       s.status,
       s.event,
       s.sql_id,
       s.prev_sql_id,
       to_char(s.logon_time,'yyyy-mm-dd hh24:mi:ss') logon_time,
       s.last_call_et
from gv$process p
join gv$session s
  on s.inst_id = p.inst_id
 and s.paddr = p.addr
where p.spid = '128698';
```

输出：

```text
INST_ID SPID   SID SERIAL# USERNAME OSUSER MACHINE                   PROGRAM       STATUS EVENT
1       128698 964 27139   APP_USER_B       user_c   DOMAIN\DEV-CLIENT-07 plsqldev.exe ACTIVE HS message to agent
```

SQL：

```text
begin :id := sys.dbms_transaction.local_transaction_id; end;
```

断：

```sql
alter system kill session '964,27139,@1' immediate;
```

然后回 `gateway01` 清对应残留：

```bash
kill -9 181095
```

`10.10.20.13` 的处理也是同样思路：

```bash
ss -antp | grep '45647'
ps -fp 查到的PID
```

如果是 Oracle，就按 `gv$session` 查 `SID/SERIAL#/INST_ID` 后 kill session。

## 结果

一开始：

```text
load average: 110.83, 110.50, 110.41
CPU idle: 0%
dg4msql 总数: 135
dg4msqlgw_alpha: 85 个，CPU 125.6
```

主要清理过程后：

```text
dg4msqlgw_alpha 从 85 个降到个位数
gw_beta/gw_gamma 的孤儿进程也被清理
load 从 110 级别降到个位数附近
CPU idle 开始恢复
```

中间观察到：

```text
top - 15:24:51
load average: 2.06, 3.60, 12.40
Cpu(s): 50.8%us, 1.5%sy, 47.6%id
```

当时还剩两个历史长连接在吃单核，继续按来源机器查掉。

这次事故的根因不是 某 ERP 系统 服务本身，也不是透明网关“配置就是高 CPU”，而是大量人工客户端长期挂着 DBLink/透明网关查询：

```text
PL/SQL Developer
DataGrip
SELECT * FROM xxx@dblink_zeta / @dblink_alpha / @DBLINK_GAMMA
HS message to agent
CLOSE_WAIT / NO_TCP / NO_CLIENT_TO_SQLSERVER 残留
```

这些会话从 2025 年一直堆到 2026 年，最后把透明网关服务器 CPU 打满。

## 后续建议

1. 不要让个人客户端直接用生产账号长期查询远程 DBLink 大表。
2. 对 `HS message to agent` 且 `last_call_et` 超过阈值的会话做定期巡检。
3. 对透明网关服务器上的 `dg4msql` 做进程级巡检，重点看：
   - `CLOSE_WAIT`
   - `NO_TCP`
   - 没有客户端只剩 SQL Server 连接
   - 运行时间超过 1 天或 7 天
4. 开发人员查询远程 SQL Server 表时，不要直接 `select * from 大表@dblink`。
5. 生产库人工查询最好加审计，至少能知道谁、哪台机器、哪个工具、哪条 SQL。

## 附录一：几个概念

### dg4msql 是什么

`dg4msql` 是 Oracle Database Gateway for SQL Server 的 agent 进程。

当 Oracle 通过 DBLink 访问 SQL Server 时，链路大概是：

```text
Oracle 源库 session
-> DBLink
-> 透明网关 listener
-> dg4msql
-> SQL Server
```

一个 Oracle session 可能拉起一个 gateway agent。会话不释放，agent 就可能一直在。

### HS message to agent

`HS message to agent` 是 Oracle Heterogeneous Services 的等待事件。

简单理解就是 Oracle session 正在等透明网关 agent 返回结果。正常情况下这个等待不应该持续几天、几十天、几百天。

如果看到：

```text
event = HS message to agent
last_call_et 很大
program = plsqldev.exe / DataGrip
```

基本就要查人工工具是不是挂了远程查询。

### CLOSE_WAIT

`CLOSE_WAIT` 表示对端已经关闭连接，本机进程还没有关闭 socket。

在这次问题里：

```text
10.10.10.10:1521 -> 10.10.20.10:xxxx (CLOSE_WAIT)
```

说明源库那边已经断了，但 `gateway01` 上 `dg4msql` 没退出。

### NO_TCP

`NO_TCP` 不是系统状态，是这次排查里自定义的分类。

意思是：

```text
lsof -p PID -iTCP 没有任何输出
```

但这个 `dg4msql` 还在跑 CPU。它基本已经没有业务连接价值。

### NO_CLIENT_TO_SQLSERVER

这个也是本次排查里自定义的分类。

意思是：

```text
没有 gateway01:1521 -> 来源机器 的连接
但还有 gateway01:随机端口 -> SQL Server:1433 的连接
```

也就是客户端侧没了，只剩网关 agent 连着 SQL Server。

## 附录二：DBA 操作手册

### 1. 网关服务器按 SID 统计 dg4msql

```bash
ps -eo pid,pcpu,args | awk '/[d]g4msql/ {cnt[$3]++; cpu[$3]+=$2} END {for(k in cnt) printf "%8.1f %5d %s\n", cpu[k], cnt[k], k}' | sort -nr
```

### 2. 看高 CPU dg4msql 明细

```bash
for p in $(ps -eo pid,pcpu,args --sort=-pcpu | awk '/[d]g4msql/ {print $1}' | head -30); do
  echo "===== PID $p ====="
  ps -o pid,lstart,etime,pcpu,args -p "$p"
  lsof -Pan -p "$p" -iTCP 2>/dev/null
done
```

### 3. 按来源 IP 分类

```bash
for p in $(pgrep -f 'dg4msql'); do
  sid=$(ps -o args= -p "$p" | awk '{print $1}')
  cpu=$(ps -o pcpu= -p "$p" | awk '{print $1}')
  etime=$(ps -o etime= -p "$p" | awk '{print $1}')
  tcp=$(lsof -Pan -p "$p" -iTCP 2>/dev/null)

  src=$(echo "$tcp" | awk '/:1521->/ {peer=$9; sub(/^.*->/,"",peer); split(peer,a,":"); print a[1]; exit}')
  dst=$(echo "$tcp" | awk '/:1433/ {peer=$9; sub(/^.*->/,"",peer); split(peer,a,":"); print a[1]; exit}')
  state=$(echo "$tcp" | awk '/:1521->/ {print $NF; exit}')

  [ -z "$src" ] && src="NO_CLIENT"
  [ -z "$dst" ] && dst="NO_SQLSERVER"
  [ -z "$state" ] && state="NO_CLIENT_STATE"

  echo "$src $sid $dst $state $cpu $etime $p"
done | awk '
{
  key=$1" "$2" -> "$3" "$4
  cnt[key]++
  cpu[key]+=$5
}
END {
  for (k in cnt) printf "%8.1f %5d %s\n", cpu[k], cnt[k], k
}
' | sort -nr
```

### 4. 来源服务器按端口反查进程

例如网关上看到：

```text
10.10.10.10:1521 -> 10.10.20.21:55655
```

去 `10.10.20.21` 上查：

```bash
ss -antp | grep '10.10.10.10:1521' | grep ':55655'
ps -fp 查到的PID
```

### 5. Oracle 源库查 session

```sql
set lines 260 pages 100
col username format a18
col osuser format a18
col machine format a35
col program format a45
col module format a45
col action format a35
col event format a45
col logon_time format a19

select p.inst_id,
       p.spid,
       s.sid,
       s.serial#,
       s.username,
       s.osuser,
       s.machine,
       s.program,
       s.module,
       s.action,
       s.status,
       s.event,
       s.sql_id,
       s.prev_sql_id,
       to_char(s.logon_time,'yyyy-mm-dd hh24:mi:ss') logon_time,
       s.last_call_et
from gv$process p
join gv$session s
  on s.inst_id = p.inst_id
 and s.paddr = p.addr
where p.spid = 'OS_PID';
```

### 6. 查 SQL 文本

```sql
select s.sid,
       s.serial#,
       p.spid,
       nvl(s.sql_id, s.prev_sql_id) sql_id,
       dbms_lob.substr(q.sql_fulltext, 3000, 1) sql_text
from gv$process p
join gv$session s
  on s.inst_id = p.inst_id
 and s.paddr = p.addr
left join gv$sqlarea q
  on q.inst_id = s.inst_id
 and q.sql_id = nvl(s.sql_id, s.prev_sql_id)
where p.spid = 'OS_PID';
```

### 7. 从 Oracle 侧 kill session

单实例：

```sql
alter system kill session 'SID,SERIAL#' immediate;
```

RAC：

```sql
alter system kill session 'SID,SERIAL#,@INST_ID' immediate;
```

如果还残留：

```sql
alter system disconnect session 'SID,SERIAL#' immediate;
```

### 8. 清网关上的残留 agent

只清 `CLOSE_WAIT`：

```bash
for p in $(pgrep -f 'dg4msql'); do
  if lsof -Pan -p "$p" -iTCP 2>/dev/null | grep -q 'CLOSE_WAIT'; then
    echo $p
  fi
done > /tmp/kill_gateway_close_wait.pids

xargs -r kill -9 < /tmp/kill_gateway_close_wait.pids
```

只清没有 TCP 连接的孤儿：

```bash
for p in $(pgrep -f 'dg4msql'); do
  if ! lsof -Pan -p "$p" -iTCP >/dev/null 2>&1; then
    echo $p
  fi
done > /tmp/kill_gateway_no_tcp.pids

xargs -r kill -9 < /tmp/kill_gateway_no_tcp.pids
```

### 9. 验证

```bash
ps -eo pid,pcpu,args | awk '/[d]g4msql/ {cnt[$3]++; cpu[$3]+=$2} END {for(k in cnt) printf "%8.1f %5d %s\n", cpu[k], cnt[k], k}' | sort -nr
top
ss -antp | grep ':1521'
```

最后一句：透明网关 CPU 打满时，不要只盯 SQL Server，也不要只重启应用。先把 `dg4msql -> 来源 IP -> Oracle session -> 客户端工具 -> SQL` 这条链路串起来，基本就不会查偏。
