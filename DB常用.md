# DB常用



## 查询ASM磁盘组空间

```sql
-- 查看ASM磁盘组总空间和剩余空间
SELECT 
    name AS "磁盘组名称",
    total_mb AS "总大小(MB)",
    free_mb AS "剩余空间(MB)",
    usable_file_mb AS "可用文件空间(MB)",
    ROUND((1 - free_mb/total_mb) * 100, 2) AS "使用率(%)"
FROM v$asm_diskgroup
ORDER BY (1 - free_mb/total_mb) DESC;

-- 查看ASM磁盘详细信息
SELECT 
    group_number AS "磁盘组号",
    disk_number AS "磁盘号",
    name AS "磁盘名称",
    total_mb AS "总大小(MB)",
    free_mb AS "剩余空间(MB)",
    ROUND((1 - free_mb/total_mb) * 100, 2) AS "使用率(%)",
    state AS "状态"
FROM v$asm_disk
ORDER BY group_number, disk_number;
```





## 查看表空间使用情况



```sql
-- 查看表空间使用情况
SELECT
    df.tablespace_name,
    df.bytes/1024/1024 "总大小(MB)",
    (df.bytes-fs.bytes)/1024/1024 "已使用(MB)",
    fs.bytes/1024/1024 "剩余空间(MB)",
    ROUND(((df.bytes-fs.bytes)/df.bytes)*100,2) "使用率(%)"
FROM
    (SELECT tablespace_name, SUM(bytes) bytes FROM dba_data_files GROUP BY tablespace_name) df,
    (SELECT tablespace_name, SUM(bytes) bytes FROM dba_free_space GROUP BY tablespace_name) fs
WHERE
    df.tablespace_name = fs.tablespace_name
ORDER BY
    ((df.bytes-fs.bytes)/df.bytes) DESC;

-- 查看特定表空间(STKHISDATA)的详细信息和数据文件
SELECT 
    file_name, 
    bytes/1024/1024 "大小(MB)", 
    autoextensible "自动扩展", 
    maxbytes/1024/1024 "最大大小(MB)",
    increment_by * 8192/1024/1024 "扩展增量(MB)"
FROM 
    dba_data_files 
WHERE 
    tablespace_name = 'ACTDATA';
```



## 查询表空间中明细的表占用

```sql
-- 查看BARDATA表空间中的所有对象
SELECT 
    owner AS "所有者",
    segment_name AS "对象名",
    segment_type AS "对象类型",
    tablespace_name AS "表空间",
    bytes/1024/1024 AS "大小(MB)"
FROM dba_segments 
WHERE tablespace_name = 'BARDATA'
ORDER BY bytes DESC;


-- 查看BARDATA表空间中的所有对象
SELECT 
    owner AS "所有者",
    segment_name AS "对象名",
    segment_type AS "对象类型",
    tablespace_name AS "表空间",
    bytes/1024/1024 AS "大小(MB)"
FROM dba_segments 
WHERE tablespace_name = 'BARDATA'
ORDER BY bytes DESC;

-- 查看BARDATA表空间的数据文件
SELECT 
    file_name,
    bytes/1024/1024 AS "大小(MB)",
    status,
    autoextensible
FROM dba_data_files 
WHERE tablespace_name = 'BARDATA';
```





## 增加表空间ASM文件

```sql
-- 初始10G 自增步进1G 最大31G（综合数据库配置的参数，单个文件最大31.768G）
ALTER TABLESPACE STKHISDATA ADD DATAFILE '+DATA' SIZE 10G AUTOEXTEND ON NEXT 1G MAXSIZE 31G;
```



## 修改ASM自增步进

```sql
-- 查询ASM文件
SELECT 
    tablespace_name,
    file_name,
    bytes/1024/1024 MB,
    autoextensible,
    increment_by,
    maxbytes/1024/1024 MB
FROM dba_data_files
WHERE file_name LIKE '+%stkdata%'  -- ASM文件
ORDER BY tablespace_name;

-- 修改自增步进（操作会影响表空间，挑使用量少的时候进行操作）
ALTER DATABASE 
DATAFILE '+DATA/mzmmdb/datafile/stkdata.321.1179154267'
AUTOEXTEND ON NEXT 1G MAXSIZE 31G;

```





##   查询活动的会话并杀掉

* 查询

  ```sql
  -- 查看当前活动的会话和正在执行的SQL
  SELECT s.sid, s.serial#, s.username, s.status, s.osuser, s.machine, 
         s.program, s.module, s.event, s.wait_time, s.seconds_in_wait,
         sq.sql_text
  FROM v$session s
  LEFT JOIN v$sql sq ON s.sql_id = sq.sql_id
  WHERE s.type != 'BACKGROUND'
  AND s.status = 'ACTIVE'
  ORDER BY s.seconds_in_wait DESC;
  
  -- 或者更简单的查询
  SELECT sid, serial#, username, status, event, sql_id
  FROM v$session
  WHERE status = 'ACTIVE'
  AND type != 'BACKGROUND';
  ```

* kill

  ```sql
  -- 使用ALTER SYSTEM KILL SESSION
  ALTER SYSTEM KILL SESSION 'SID,SERIAL#' IMMEDIATE;
  
  -- 例如，如果SID=123，SERIAL#=1234
  ALTER SYSTEM KILL SESSION '123,1234' IMMEDIATE;
  ```




## 服务器信息

* SYCRM： 
    * 10.1.90.73
        端口：22
        账号：root
        密码：fa%#fdjasE5!
    * 同步MM生产数据到这里，事业部查数据走这里，分散MM生产库压力
* PDA：
    * ![image-20250314111405884](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202507140928294.png)
    * 业务量不高、金币联盟积分等数据跑这里



## 临时表目录 ASM 转换为 文件系统路径

* 查看现存临时文件

    ```sql
    -- 查看现有临时文件
    SELECT file#, name, status FROM v$tempfile;
    ```

* 修改数据库参数

    ```sql
    -- 修改默认临时文件创建路径
    ALTER SYSTEM SET db_create_file_dest='/u01/app/oracle/oradata/yxtest/tempfile' SCOPE=SPFILE;
    
    -- 重启使参数生效
    SHUTDOWN IMMEDIATE;
    STARTUP;
    ```

* 迁移现存临时表（如需要的话，一般临时表都是Oracle自动管理的）

    **一般这个时候，再查询现有临时文件，Oracle已经自动配置好了，如果没有的话，再通过下边步骤手动创建**

    ```sql
    -- 第一步：创建新的临时表空间（文件系统）
    CREATE TEMPORARY TABLESPACE TEMP_NEW TEMPFILE '/u01/app/oracle/oradata/yxtest/tempfile/temp_new.dbf' SIZE 2G AUTOEXTEND ON;
    
    -- 第二步：切换默认临时表空间
    ALTER DATABASE DEFAULT TEMPORARY TABLESPACE TEMP_NEW;
    
    -- 第三步：验证存储配置
    SELECT file_name, tablespace_name 
    FROM dba_temp_files;
    ```

* 迁移后确认

    ```sql
    -- 确认新临时表空间使用情况
    SELECT tablespace_name, bytes_used, bytes_free 
    FROM v$temp_space_header;
    ```




## 查找表上的锁

```sql
    SELECT 
    l.sid,
    s.serial#,
    s.username,
    s.machine,
    s.program,
    s.osuser,
    l.type lock_type,
    CASE l.lmode
        WHEN 0 THEN 'None'
        WHEN 1 THEN 'Null'
        WHEN 2 THEN 'Row-S (SS)'
        WHEN 3 THEN 'Row-X (SX)'
        WHEN 4 THEN 'Share'
        WHEN 5 THEN 'S/Row-X (SSX)'
        WHEN 6 THEN 'Exclusive'
    END lock_mode,
    q.sql_text current_sql,
    q_prev.sql_text prev_sql,
    o.object_name
FROM 
    v$lock l
LEFT JOIN v$session s ON l.sid = s.sid
LEFT JOIN dba_objects o ON l.id1 = o.object_id
LEFT JOIN v$sql q ON s.sql_id = q.sql_id
LEFT JOIN v$sql q_prev ON s.prev_sql_id = q_prev.sql_id
WHERE 
    o.object_name = 'FUNMENU'

```



## SHOW CREATE TABLE

```sql
SELECT DBMS_METADATA.GET_DDL('TABLE', '<表名>') FROM dual;
```



## 实例缩配置

**1. 内存参数检查**

**关键参数**

```sql:工作资料/文档/Oracle 运维常用.md
-- 查看当前内存配置
SHOW PARAMETER sga_target
SHOW PARAMETER pga_aggregate_target
SHOW PARAMETER memory_target
```

**典型问题**

- **SGA_MAX_SIZE** > 可用物理内存
- **PROCESSES** 参数超过新CPU核心数限制

**解决方案**

```sql
-- 调整示例（16G内存环境）
ALTER SYSTEM SET sga_max_size=8G SCOPE=spfile;
ALTER SYSTEM SET pga_aggregate_target=4G SCOPE=spfile;
ALTER SYSTEM SET processes=500 SCOPE=spfile;  -- 8核建议值
```

**2. 进程数限制**

**检查点**

```sql
-- 查看当前配置
SHOW PARAMETER processes
SHOW PARAMETER sessions
```

**计算规则**

- **processes** = (CPU核心数 * 50) + 50 → 8核应 ≤ 450
- **sessions** = processes * 1.1 + 5

**3. 存储配置验证**

**ASM检查（如使用）**

```bash:工作资料/文档/圣元 Oracle 测试库定时同步备份库操作步骤.md
# 检查ASM磁盘组状态
asmcmd lsdg
```

**文件系统检查**

```sql
-- 查看数据文件状态
SELECT name, status FROM v$datafile WHERE status != 'ONLINE';
```

**4. 日志文件限制**

**关键位置**

```sql
-- 检查redo日志大小
SELECT group#, bytes/1024/1024 "MB", status FROM v$log;

-- 检查控制文件
SELECT * FROM v$controlfile;
```

**典型问题**

- Redo日志文件过大（如2G）导致16G内存不足

**5. 资源管理器配置**

**检查项**

```sql
-- 查看资源计划
SELECT * FROM v$rsrc_plan;
```

**解决方案**

```sql
-- 禁用资源管理器
ALTER SYSTEM SET resource_manager_plan='' SCOPE=both;
```

**6. ASMM自动内存管理**

**配置建议**

```sql
-- 16G内存推荐配置
ALTER SYSTEM SET memory_target=0 SCOPE=spfile;  -- 禁用AMM
ALTER SYSTEM SET sga_target=8G SCOPE=spfile;
ALTER SYSTEM SET pga_aggregate_target=4G SCOPE=spfile;
```

**7. 操作系统参数**

**关键内核参数**

```bash
# 检查当前值
sysctl -a | grep -E 'shmmax|shmmni|sem|file-max'

# 16G内存推荐值
echo "
kernel.shmmax = 8589934592  # 8G
kernel.shmall = 2097152     # 8G/page_size(通常4K)
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
fs.file-max = 6815744
" >> /etc/sysctl.conf
```

**8. 集群配置（如RAC）**

```bash:工作资料/文档/圣元 Oracle 测试库定时同步备份库操作步骤.md
# 检查集群状态
crsctl check crs
```

**典型问题**

- 节点驱逐（Node eviction）导致无法启动

**9. 启动日志分析**

**关键日志位置**

```bash
# 查看alert日志
tail -100f $ORACLE_BASE/diag/rdbms/${ORACLE_SID}/${ORACLE_SID}/trace/alert_${ORACLE_SID}.log
```

**常见错误模式**

- **ORA-00845**: /dev/shm 不足 → `mount -o remount,size=8G /dev/shm`
- **ORA-27102**: 共享内存不足 → 调整shmmax
- **ORA-00068**: 参数值无效 → 检查processes/sessions

**10. 分步启动测试**

```bash
# 1. 仅启动到nomount
STARTUP NOMOUNT

# 2. 挂载控制文件
ALTER DATABASE MOUNT;

# 3. 分阶段打开
ALTER DATABASE OPEN;
```



## 查询SEQUENCE和修改 NEXTVAL 和 步长

```sql
-- 举个例子 LABT_SEQUENCE 
ALTER SEQUENCE LABT_SEQUENCE INCREMENT BY 999;
SELECT LABT_SEQUENCE.NEXTVAL FROM DUAL;  -- 此时返回2000
ALTER SEQUENCE LABT_SEQUENCE INCREMENT BY 1;
-- 立即验证
SELECT LABT_SEQUENCE.CURRVAL FROM DUAL;  -- 应返回2000
```



