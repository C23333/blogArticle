---
title: 服务器 Tomcat 单机单实例 改 多实例软负载均衡
date: 2025-04-21 05:21:06
tags:
  - 服务器
  - Tomcat
keywords: Tomcat单机多节点
categories: 实操
---



## 服务器 Tomcat 单机单实例 改 多实例软负载均衡

> 操作日期：2025/03/11
>
> 记录日期：2025/03/11
>
> **文尾我给出了我Nginx和Tomcat完整的配置文件，可以看看（Tomcat只修改了server.xml）**
>
> **部分数据因安全考虑未给出**

### 一、操作前环境确认

* 系统内核：CentOS Linux release 7.2.1511 (Core) 
* 硬件配置：12C16G
* 软件配置：
  * Tomcat路径：/xxx/www/apache-tomcat-7.0.88
  * 未安装Nginx
  * Tomcat版本：apache-tomcat-7.0.88



### 二、多实例配置、Nginx软负载均衡配置



#### **1. 备份现有环境**
- **备份Tomcat目录**：  
  
  ```bash
  cp -r xxxwww/apache-tomcat-7.0.88 xxxwww/apache-tomcat-7.0.88_backup
  ```

#### **2. 确认现有服务状态**
- **检查Tomcat运行状态**：  
  ```bash
  ps -ef | grep tomcat
  netstat -tunlp | grep java  # 确认当前监听的端口（如80、443、8009等）
  ```



#### **3. 创建实例目录**
```bash
# 创建3个实例目录（tomcat_instance1、tomcat_instance2、tomcat_instance3）
mkdir -p xxxwww/tomcat_instance/{instance1,instance2,instance3}

# 复制配置文件（避免污染原Tomcat）
cp -r xxxwww/apache-tomcat-7.0.88/conf xxxwww/tomcat_instance/instance1/
cp -r xxxwww/apache-tomcat-7.0.88/conf xxxwww/tomcat_instance/instance2/
cp -r xxxwww/apache-tomcat-7.0.88/conf xxxwww/tomcat_instance/instance3/
```

#### **4. 修改端口配置**
- **实例1配置（`/xxx/www/tomcat_instance/instance1/conf/server.xml`）**：  

  ```xml
  <?xml version='1.0' encoding='utf-8'?>
  <Server port="8006" shutdown="SHUTDOWN">
    // ... 保持所有Listener配置不变 ...
    <GlobalNamingResources>
      // ... 保持资源配置不变 ...
    </GlobalNamingResources>
  
    // 这步的   proxyPort="443" scheme="https" secure="true" 巨重要，一定要搞上啊，不然你看请求走HTTPS，Tomcat返回你HTTP，哭都没地 
    <Service name="Catalina">
      <Connector port="8081" protocol="HTTP/1.1" 
                 connectionTimeout="20000" 
                 redirectPort="443"  proxyPort="443" scheme="https" secure="true" />
                 
      <Connector port="8010" protocol="AJP/1.3" 
                 redirectPort="443"
                 secretRequired="true"
                 secret="your_secure_password" />
  
      <Engine name="Catalina" defaultHost="localhost">
        <Valve className="org.apache.catalina.valves.AccessLogValve"
               directory="xxxwww/tomcat_instance/instance1/logs"
               prefix="instance1_access" 
               suffix=".log"
               pattern="%h %l %u %t &quot;%r&quot; %s %b %D" />
        // ... 保持Realm配置不变 ...
  ```

- **实例2配置（`/xxx/www/tomcat_instance/instance2/conf/server.xml`）**：  

  ```xml
  <?xml version='1.0' encoding='utf-8'?>
  <Server port="8007" shutdown="SHUTDOWN">
    // ... 保持所有Listener配置不变 ...
    <GlobalNamingResources>
      // ... 保持资源配置不变 ...
    </GlobalNamingResources>
  
    <Service name="Catalina">
      <Connector port="8082" protocol="HTTP/1.1" 
                 connectionTimeout="20000" 
                 redirectPort="443"   proxyPort="443" scheme="https" secure="true"/>
                 
      <Connector port="8011" protocol="AJP/1.3" 
                 redirectPort="443"
                 secretRequired="true"
                 secret="your_secure_password" />
  
      <Engine name="Catalina" defaultHost="localhost">
        <Valve className="org.apache.catalina.valves.AccessLogValve"
               directory="xxxwww/tomcat_instance/instance2/logs"
               prefix="instance1_access" 
               suffix=".log"
               pattern="%h %l %u %t &quot;%r&quot; %s %b %D" />
        // ... 保持Realm配置不变 ...
  ```

- **实例3配置（`/xxx/www/tomcat_instance/instance3/conf/server.xml`）**：  

  ```xml
  <?xml version='1.0' encoding='utf-8'?>
  <Server port="8008" shutdown="SHUTDOWN">
    // ... 保持所有Listener配置不变 ...
    <GlobalNamingResources>
      // ... 保持资源配置不变 ...
    </GlobalNamingResources>
  
    <Service name="Catalina">
      <Connector port="8083" protocol="HTTP/1.1" 
                 connectionTimeout="20000" 
                 redirectPort="443"   proxyPort="443" scheme="https" secure="true"/>
                 
      <Connector port="8012" protocol="AJP/1.3" 
                 redirectPort="443"
                 secretRequired="true"
                 secret="your_secure_password" />
  
      <Engine name="Catalina" defaultHost="localhost">
        <Valve className="org.apache.catalina.valves.AccessLogValve"
               directory="xxxwww/tomcat_instance/instance3/logs"
               prefix="instance1_access" 
               suffix=".log"
               pattern="%h %l %u %t &quot;%r&quot; %s %b %D" />
        // ... 保持Realm配置不变 ...
  ```

#### **5. 配置实例管理脚本**
- 封装了个小脚本
  
  ```bash
  #!/bin/bash
  CATALINA_HOME="xxxwww/apache-tomcat-7.0.88"
  INSTANCE_ROOT="xxxwww/tomcat_instance"
  LOG_FILE="/var/log/tomcat_cluster.log"
  
  log() {
      echo "$(date '+%Y-%m-%d %H:%M:%S') $@" | tee -a $LOG_FILE
  }
  
  start_instance() {
      local instance_name=$1
      local instance_path="$INSTANCE_ROOT/$instance_name"
      
      if [ ! -d "$instance_path" ]; then
          log "[ERROR] 实例目录不存在: $instance_path"
          return 1
      fi
  
      local pid=$(get_pid $instance_name)
      if [ -n "$pid" ]; then
          log "[WARN] 实例 $instance_name 已在运行 (PID: $pid)"
          return 0
      fi
  
      log "[INFO] 正在启动实例 $instance_name"
      export CATALINA_BASE="$instance_path"
      export CATALINA_PID="$instance_path/bin/catalina.pid"
      
      $CATALINA_HOME/bin/startup.sh >/dev/null 2>&1
      sleep 2
      
      if [ -f "$CATALINA_PID" ]; then
          log "[SUCCESS] 实例 $instance_name 启动成功 (PID: $(cat $CATALINA_PID))"
      else
          log "[ERROR] 实例 $instance_name 启动失败"
          return 1
      fi
  }
  
  stop_instance() {
      local instance_name=$1
      local instance_path="$INSTANCE_ROOT/$instance_name"
      
      if [ ! -d "$instance_path" ]; then
          log "[ERROR] 实例目录不存在: $instance_path"
          return 1
      fi
  
      local pid=$(get_pid $instance_name)
      if [ -z "$pid" ]; then
          log "[WARN] 实例 $instance_name 未运行"
          return 0
      fi
  
      log "[INFO] 正在停止实例 $instance_name (PID: $pid)"
      export CATALINA_BASE="$instance_path"
      export CATALINA_PID="$instance_path/bin/catalina.pid"
      
      $CATALINA_HOME/bin/shutdown.sh -force >/dev/null 2>&1
      sleep 2
      
      if ps -p $pid >/dev/null; then
          kill -9 $pid
          log "[WARN] 强制终止实例 $instance_name (PID: $pid)"
      fi
      
      [ -f "$CATALINA_PID" ] && rm -f "$CATALINA_PID"
      log "[SUCCESS] 实例 $instance_name 已停止"
  }
  
  get_pid() {
      ps -ef | grep "catalina.base=$INSTANCE_ROOT/$1" | grep -v grep | awk '{print $2}'
  }
  
  case $1 in
      start-all)
          for instance in $(ls $INSTANCE_ROOT); do
              start_instance $instance
          done
          ;;
      start-instance)
          start_instance $2
          ;;
      stop-all)
          for instance in $(ls $INSTANCE_ROOT); do
              stop_instance $instance
          done
          ;;
      stop-instance)
          stop_instance $2
          ;;
      restart-all)
          $0 stop-all
          sleep 2
          $0 start-all
          ;;
      restart-instance)
          stop_instance $2
          sleep 2
          start_instance $2
          ;;
      status)
          echo "运行中的实例:"
          ps -ef | grep "catalina.base=$INSTANCE_ROOT" | grep -v grep | awk '{print $NF" (PID:"$2")"}' | sort
          ;;
      *)
          echo "使用方法: $0 {start-all|start-instance 实例名|stop-all|stop-instance 实例名|restart-all|restart-instance 实例名|status}"
          exit 1
  esac
  ```

#### 6.复制webapps文件、创建需要目录

```bash
# 同步webapps目录并排除指定日志文件
sudo rsync -avz \
--exclude='*/logs/raqsoftReport_*.log' \
--exclude='temp/' \
--exclude='work/' \
xxxwww/apache-tomcat-7.0.88/webapps/v5demo* \
xxxwww/tomcat_instance/instance<x>/webapps/

# 创建需要的目录
mkdir xxxwww/tomcat_instance/instance<x>/logs
```



#### **7. 启动实例并验证**

```bash
# 启动实例
xxxwww/tomcat_control.sh start-all

# 验证端口监听
netstat -tunlp | grep java  # 查看端口是不是都监听上了 
curl http://localhost:<port>  # 检查应用是否正常响应
```

---

### **三、配置Nginx负载均衡**
#### **1. 安装Nginx**
```bash
sudo yum install epel-release -y
sudo yum install nginx -y
# 停止原 Tomcat 服务器，避免端口占用导致Nginx启动失败
xxxwww/apache-tomcat-7.0.88/shutdown.sh
sudo systemctl start nginx
sudo systemctl enable nginx
```

#### **2. 配置反向代理**

* 复制证书

  **如果你有的选，别犹豫，去官网下个证书文件，转换好多问题，九九八十一难**
  
  ```bash
  # 查看原Tomcat证书路径
  grep 'keystoreFile' xxxwww/apache-tomcat-7.0.88/conf/server.xml
  # 输出类似：keystoreFile="conf/xxxxxxxx.jks"
  
  # 推荐证书存放路径（Nginx标准位置）：
  mkdir -p /etc/nginx/ssl
  cp xxxwww/apache-tomcat-7.0.88/conf/xxxxxxxx.jks /etc/nginx/ssl/
  cd /etc/nginx/ssl
  # 转换证书格式
  keytool -importkeystore -J-Dkeystore.pkcs12.legacy -srckeystore xxxxxxxx.jks -destkeystore xxxxxxxx.p12 -srcstoretype JKS -deststoretype PKCS12 -alias "_.xxxxxxxx.com" -srcstorepass 1234 -deststorepass 123456 -destkeypass 123456 # 新协议强制要求密码 ≥ 6位
  # 检查PKCS12文件完整性
  openssl pkcs12 -info -in xxxxxxxx.p12 -password pass:123456 -nodes | head -n 20
  
  # 确认包含完整证书链
  openssl pkcs12 -in xxxxxxxx.p12 -password pass:123456 -nokeys | grep 'subject='
  # 应显示两个证书条目
  cp xxxxxxxx.jks xxxxxxxx.jks.bak_$(date +%Y%m%d%H%M)
  cp xxxxxxxx.p12 xxxxxxxx.p12.bak_$(date +%Y%m%d%H%M)
  
  # 转换证书（使用绝对路径）
  sudo openssl pkcs12 -in xxxxxxxx.p12 -clcerts -nokeys -out xxxxxxxx.crt -passin pass:123456
  sudo openssl pkcs12 -in xxxxxxxx.p12 -nocerts -nodes -out xxxxxxxx.key -passin pass:123456
  
  # 自动获取中间证书
  openssl s_client -connect rqbb.xxxxxxxx.com:443 -servername rqbb.xxxxxxxx.com -showcerts 2>&1 < /dev/null | awk '/BEGIN CERTIFICATE/{i++}i==2' | sudo tee /etc/nginx/ssl/intermediate.crt
  
  # 验证中间证书
  sudo openssl x509 -in /etc/nginx/ssl/intermediate.crt -text -noout | grep -E 'Issuer:|Subject:'
  
  # 设置权限
  sudo chmod 644 /etc/nginx/ssl/intermediate.crt
  sudo chown nginx:nginx /etc/nginx/ssl/intermediate.crt
  ```
  
  

- **创建负载均衡配置文件（`/etc/nginx/conf.d/tomcat_lb.conf`）**：  
  
  ```nginx
  upstream tomcat_cluster {
      server 127.0.0.1:8080 weight=1;  # 实例1
      server 127.0.0.1:8081 weight=1;  # 实例2
      ip_hash; # 会话保持（测试的时候可以注释掉，多次curl看看是不是负载到不同的端口上）
  }
  
  server {
      listen 80;
      server_name rqbb.xxxxxxxx.com;
  
      # HTTP强制跳转HTTPS（若需保留HTTP，则注释下一行，打开location块）
      return 301 https://$host$request_uri;
      
  #    location / {
  #        proxy_pass http://tomcat_cluster;
  #        proxy_set_header Host $host;
  #        proxy_set_header X-Real-IP $remote_addr;
  #        proxy_set_header X-Forwarded-Proto $scheme;
  #        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  #    }
  }
  
  # SSL配置（由Nginx统一处理HTTPS）
  server {
      listen 443 ssl;
      server_name rqbb.xxxxxxxx.com
  
      ssl_certificate /path/to/cert.pem;
      ssl_certificate_key /path/to/privkey.pem;
  
      location / {
          proxy_pass http://tomcat_cluster;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-Proto $scheme;
      }
  }
  ```

#### **3. 重启Nginx并测试**
```bash
sudo nginx -t          # 检查配置语法
sudo systemctl reload nginx  # 平滑重启

# 测试负载均衡
curl http://localhost   # 观察请求是否分发到不同实例
```

---

### **四、验证与监控**
#### **1. 功能验证**
- **流量分发**：多次访问 `http(s)://服务器IP`，检查不同实例的日志（`logs/localhost_access_log.*`）。  
- **HTTPS**：访问 `https://域名`，确保证书有效且无告警。  

#### **2. 监控关键指标**
- **Nginx状态**：  
  ```bash
  watch -n 1 "sudo netstat -tunlp | grep nginx"  # 监控连接数
  tail -f /var/log/nginx/access.log              # 实时请求日志
  ```

- **Tomcat实例状态**：  
  ```bash
  tail -f xxxwww/tomcat_instance/instance1/logs/catalina.out
  ```

---

### **五、回滚方案**

**留了一手，要是没搞定，就走控制脚本，停止所有集群实例，停止Nginx，然后重启旧的Tomcat**

#### **1. 快速回滚条件**
- 新实例或Nginx配置异常，导致服务不可用。  
- 负载均衡策略不符合预期。  

#### **2. 回滚步骤**
- **停止新实例和Nginx**：  
  
  ```bash
  ./tomcat_control.sh stop-all
  service nginx stop
  ```
  
- **恢复原Tomcat配置**：  
  
  ```bash
  # 重启原Tomcat（如果已停止）
  xxxwww/apache-tomcat-7.0.88/bin/shutdown.sh
  xxxwww/apache-tomcat-7.0.88/bin/startup.sh
  ```
  
- **恢复Nginx配置**(如果Nginx没配置其他web的话，这个不管也行，后边慢慢改负载均衡)：  
  
  ```bash
  cp /etc/nginx/nginx.conf.backup /etc/nginx/nginx.conf
  cp -r /etc/nginx/conf.d_backup/* /etc/nginx/conf.d/
  sudo systemctl reload nginx
  ```

---

### **六、注意事项**
1. **端口冲突**：确保新实例端口（8080/8081）未被其他服务占用。  
2. **会话保持**：若应用依赖会话（如登录态），需在Nginx中启用 `ip_hash` 或配置Tomcat会话复制。  
3. **日志分割**：建议为每个实例配置独立的日志目录，避免混杂。  
4. **防火墙**：开放Nginx监听端口（80/443）及Tomcat实例端口。  



### 七、我的Nginx和Tomcat实例配置

*  Nginx配置：

  ```bash
  # 全局SSL配置
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';
  ssl_prefer_server_ciphers on;
  ssl_session_cache shared:SSL:10m;
  ssl_session_timeout 1d;
  ssl_session_tickets off;
  ssl_stapling on;
  ssl_stapling_verify on;
  resolver 8.8.8.8 8.8.4.4 valid=300s;
  resolver_timeout 5s;
  
  # 上游服务器配置
  upstream tomcat_cluster {
      # 会话保持策略
      #ip_hash;  # 基于客户端IP
      hash $cookie_JSESSIONID consistent;  # 基于Tomcat会话ID
      
      server 127.0.0.1:8081 max_fails=3 fail_timeout=30s; # 实例1
      server 127.0.0.1:8082 max_fails=3 fail_timeout=30s; # 实例2
      server 127.0.0.1:8083 max_fails=3 fail_timeout=30s; # 实例3
      
      keepalive 32;  # 保持连接池
  }
  
  # HTTP重定向到HTTPS
  server {
      listen 80;
      server_name rqbb.xxxxxxxx.com;
      return 301 https://$host$request_uri;
  }
  
  # HTTPS服务器
  server {
      listen 443 ssl http2;
      server_name rqbb.xxxxxxxx.com;
  
      # 证书路径
      ssl_certificate     /etc/nginx/ssl/fullchain.crt;  # 服务器证书 + 中间证书
      ssl_certificate_key /etc/nginx/ssl/server.key;     # 私钥
  
      # 安全头设置
      add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
      add_header X-Content-Type-Options nosniff;
      add_header X-Frame-Options SAMEORIGIN;
      add_header X-XSS-Protection "1; mode=block";
  
      # 访问日志
      access_log /var/log/nginx/tomcat_access.log main buffer=32k;
      error_log /var/log/nginx/tomcat_error.log warn;
  
      # 代理配置
      location / {
          proxy_pass http://tomcat_cluster;
          proxy_http_version 1.1;
          proxy_set_header Connection "";
          
          # 透传必要头信息
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-Proto $scheme;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          
          # 超时设置
          proxy_connect_timeout 10s;
          proxy_read_timeout 30s;
          proxy_send_timeout 30s;
  
          proxy_buffer_size 4k;
          proxy_buffers 8 16k;
          proxy_busy_buffers_size 24k;
          
          # 错误处理
          proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
      }
  
      # 健康检查端点
      location /health {
          access_log off;
          proxy_pass http://tomcat_cluster;
          proxy_set_header X-Health-Check "true";
      }
  }
  
  ```

* 实例1配置：

  ```bash
  <?xml version='1.0' encoding='utf-8'?>
  <Server port="8006" shutdown="SHUTDOWN">
    <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
    <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
    <Listener className="org.apache.catalina.core.JasperListener" />
    <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
    <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
    <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
    <GlobalNamingResources>
      <Resource name="UserDatabase" auth="Container"
                type="org.apache.catalina.UserDatabase"
                description="User database that can be updated and saved"
                factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
                pathname="conf/tomcat-users.xml" />
    </GlobalNamingResources>
  
    <Service name="Catalina">
      <Connector port="8081" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="443" proxyPort="443" scheme="https" secure="true"/>
      <!-- <Connector port="8001" protocol="HTTP/1.1" connectionTimeout="20000" scheme="https" redirectPort="8443" /> -->
      <Connector port="8010" protocol="AJP/1.3" redirectPort="443" />
      <!-- <Connector port="443" protocol="org.apache.coyote.http11.Http11NioProtocol"  maxThreads="500" SSLEnabled="true" scheme="https" secure="true"
           clientAuth="false" sslProtocol="TLS"  
  		 ciphers="TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_256_CBC_SHA25"
  		 keystoreFile="conf/xxxxxxxx.jks" keystorePass="1234"/> -->
  
      <Engine name="Catalina" defaultHost="localhost">
        <Valve className="org.apache.catalina.valves.AccessLogValve"
               directory="xxxwww/tomcat_instance/instance1/logs"
               prefix="instance1_access" 
               suffix=".log"
               pattern="%h %l %u %t &quot;%r&quot; %s %b %D" />
        <Realm className="org.apache.catalina.realm.LockOutRealm">
          <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
                 resourceName="UserDatabase"/>
        </Realm>
        <Host name="localhost"  appBase="webapps"
              unpackWARs="true" autoDeploy="true">
          <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
                 prefix="localhost_access_log." suffix=".txt"
                 pattern="%h %l %u %t &quot;%r&quot; %s %b" />
        </Host>
      </Engine>
    </Service>
  </Server>
  
  ```

* 实例2配置：

  ```bash
  <?xml version='1.0' encoding='utf-8'?>
  <Server port="8007" shutdown="SHUTDOWN">
    <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
    <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
    <Listener className="org.apache.catalina.core.JasperListener" />
    <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
    <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
    <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
    <GlobalNamingResources>
      <Resource name="UserDatabase" auth="Container"
                type="org.apache.catalina.UserDatabase"
                description="User database that can be updated and saved"
                factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
                pathname="conf/tomcat-users.xml" />
    </GlobalNamingResources>
  
    <Service name="Catalina">
      <Connector port="8082" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="443"  proxyPort="443" scheme="https" secure="true"/>
      <!-- <Connector port="8001" protocol="HTTP/1.1" connectionTimeout="20000" scheme="https" redirectPort="8443" /> -->
      <Connector port="8011" protocol="AJP/1.3" redirectPort="443" />
      <!-- <Connector port="443" protocol="org.apache.coyote.http11.Http11NioProtocol"  maxThreads="500" SSLEnabled="true" scheme="https" secure="true"
           clientAuth="false" sslProtocol="TLS"  
  		 ciphers="TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_256_CBC_SHA25"
  		 keystoreFile="conf/xxxxxxxx.jks" keystorePass="1234"/> -->
  
      <Engine name="Catalina" defaultHost="localhost">
        <Valve className="org.apache.catalina.valves.AccessLogValve"
               directory="/xxx/www/tomcat_instance/instance2/logs"
               prefix="instance1_access" 
               suffix=".log"
               pattern="%h %l %u %t &quot;%r&quot; %s %b %D" />
        <Realm className="org.apache.catalina.realm.LockOutRealm">
          <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
                 resourceName="UserDatabase"/>
        </Realm>
        <Host name="localhost"  appBase="webapps"
              unpackWARs="true" autoDeploy="true">
          <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
                 prefix="localhost_access_log." suffix=".txt"
                 pattern="%h %l %u %t &quot;%r&quot; %s %b" />
        </Host>
      </Engine>
    </Service>
  </Server>
  
  ```

* 实例3配置：

  ```bash
  <?xml version='1.0' encoding='utf-8'?>
  <Server port="8008" shutdown="SHUTDOWN">
    <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
    <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
    <Listener className="org.apache.catalina.core.JasperListener" />
    <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
    <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
    <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
    <GlobalNamingResources>
      <Resource name="UserDatabase" auth="Container"
                type="org.apache.catalina.UserDatabase"
                description="User database that can be updated and saved"
                factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
                pathname="conf/tomcat-users.xml" />
    </GlobalNamingResources>
  
    <Service name="Catalina">
      <Connector port="8083" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="443"  proxyPort="443" scheme="https" secure="true"/>
      <!-- <Connector port="8001" protocol="HTTP/1.1" connectionTimeout="20000" scheme="https" redirectPort="8443" /> -->
      <Connector port="8012" protocol="AJP/1.3" redirectPort="443" />
      <!-- <Connector port="443" protocol="org.apache.coyote.http11.Http11NioProtocol"  maxThreads="500" SSLEnabled="true" scheme="https" secure="true"
           clientAuth="false" sslProtocol="TLS"  
  		 ciphers="TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_256_CBC_SHA25"
  		 keystoreFile="conf/xxxxxxxx.jks" keystorePass="1234"/> -->
  
      <Engine name="Catalina" defaultHost="localhost">
        <Valve className="org.apache.catalina.valves.AccessLogValve"
               directory="xxxwww/tomcat_instance/instance3/logs"
               prefix="instance1_access" 
               suffix=".log"
               pattern="%h %l %u %t &quot;%r&quot; %s %b %D" />
        <Realm className="org.apache.catalina.realm.LockOutRealm">
          <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
                 resourceName="UserDatabase"/>
        </Realm>
        <Host name="localhost"  appBase="webapps"
              unpackWARs="true" autoDeploy="true">
          <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
                 prefix="localhost_access_log." suffix=".txt"
                 pattern="%h %l %u %t &quot;%r&quot; %s %b" />
        </Host>
      </Engine>
    </Service>
  </Server>
  ```

* 旧 Tomcat 单节点的配置：

  ```bash
  <?xml version='1.0' encoding='utf-8'?>
  <Server port="8005" shutdown="SHUTDOWN">
    <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
    <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
    <Listener className="org.apache.catalina.core.JasperListener" />
    <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
    <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
    <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
    <GlobalNamingResources>
      <Resource name="UserDatabase" auth="Container"
                type="org.apache.catalina.UserDatabase"
                description="User database that can be updated and saved"
                factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
                pathname="conf/tomcat-users.xml" />
    </GlobalNamingResources>
  
    <Service name="Catalina">
      <Connector port="80" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="443" />
      <Connector port="8001" protocol="HTTP/1.1" connectionTimeout="20000" scheme="https" redirectPort="8443" />
      <Connector port="8009" protocol="AJP/1.3" redirectPort="443" />
      <Connector port="443" protocol="org.apache.coyote.http11.Http11NioProtocol"  maxThreads="500" SSLEnabled="true" scheme="https" secure="true"
           clientAuth="false" sslProtocol="TLS"  
  		 ciphers="TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_256_CBC_SHA25"
  		 keystoreFile="conf/xxxxxxxx.jks" keystorePass="1234"/>
  
      <Engine name="Catalina" defaultHost="localhost">
        <Realm className="org.apache.catalina.realm.LockOutRealm">
          <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
                 resourceName="UserDatabase"/>
        </Realm>
        <Host name="localhost"  appBase="webapps"
              unpackWARs="true" autoDeploy="true">
          <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
                 prefix="localhost_access_log." suffix=".txt"
                 pattern="%h %l %u %t &quot;%r&quot; %s %b" />
        </Host>
      </Engine>
    </Service>
  </Server>
  
   
  
  ```

  