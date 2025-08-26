---
title: Bind9搭建个人DNS服务器
date: 2023-04-07 02:29:59
tags:
  - DNS
  - Bind9
  - 自建DNS
keywords: 自建DNS
categories: 实操
---



# Bind9搭建个人DNS服务器

### 一、安装和启用一个简单的正向解析DNS服务

> 正向解析： domian -> ip
   反向解析： ip -> domain 
   因为一个 IP 可能被多个域名使用，所以在进行反向解析时要先验证一个 IP 地址是否对应一个或者多个域名。若从 IP 出发遍历整个DNS系统来验证，将会因工程浩大而无法实现。因此，RFC1035 定义了 PTR（Pointer Record）记录。PTR 记录将 IP 地址指向域名

#### 1.安装bind9
```bash
apt-get install -y bind9
apt-get install -y dnsutils   # dns测试工具，提供如nslookup、dig等dns测试分析工具
```

#### 2.配置named.conf.local文件
```bash
#切换到bind9配置目录
cd /etc/bind
```
* named.conf.\*共有三个文件：
    * `named.conf.local`：为自定义区域配置文件
    * `named.conf.options`：为DNS全局选项配置文件
    * `named.conf.default-zones`：默认区域，如localhost

* 此处我们配置`named.conf.local`文件即可，配置如下：
```bash
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";
    zone "baidu.com" {                    // baidu.com  可以替换为任意FQDN（Fully qualified domain name-完全合格的域名）。说人话就是合规的域名
        type master;                      // type       master|slave  主|从DNS服务器
        file "/etc/bind/db.baidu.com";    // file       具体解析记录文件
        //allow-update {ip|none};         // 允许哪个ip可以使用 nsupdate 动态更新区域文件
    };
```

#### 3.配置域名具体解析记录文件
* 默认bind9安装后提供了正向解析和反向解析的模板文件，我们只需要复制改动即可
        * `db.local` localhost正向区文件，用于将名字localhost转换为本地回送IP地址 (127.0.0.1)
        * `db.127` localhost反向区文件，用于将本地回送IP地址(127.0.0.1)转换为名字localhost

* 此处我们只处理正向解析
```bash
# 复制模板文件 -a -> 保留链接、文件属性，并复制目录下的所有内容。其作用等于dpR参数组合
cp -a /etc/bind/db.local /etc/bind/db.baidu.com
```
* 配置正向解析文件
```bash
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     localhost. liuyongkai.hotmail.com. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      A       10.0.2.97
@       IN      NS      localhost.
aaa     IN      A       10.0.2.97
bbb     IN      CNAME   aaa
```

**以下为文件内容说明，不想看可以直接跳到第 4 步**

* 文件内容说明：
  ```bash
  $TTL 86400 <==定义整个记录的TTL时间为86400 
  |example.com. | IN | SOA | localhost. | liuyongkai.hotmail.com.| ( 
  |     name    | IN | TYPE|   value 1  |         value 2        | 
                                2              ; Serial                  <=== value 3
                                604800         ; Refresh                 <=== value 4
                                86400          ; Retry                   <=== value 5
                                2419200        ; Expire                  <=== value 6
                                604800 )       ; Negative Cache TTL      <=== value 7
```
以上元素意义如下：
| 元素 | 意义 |
|:----:|:----:|
| name | 当前区域的名字，例如“baidu.com.” com后的点必须写，不然系统会自动将域名再补一次，变成"baidu.com.baidu.com."|
| type   | 资源类型为SOA类型 |
| value1 | 主DNS的名称，例如ns.baidu.com. |
| value2 |  DNS服务器的管理员邮箱，注意：因为@在资源记录中有特殊含义，这里用点来代替，例如liuyongkai.hotmail.com. |
| value3 | 序列号(serial number) 十进制表示，不能超过10位，通常使用日期时间戳，例如2021071501；其作用是主从同步，当主DNS服务器的解析文件发生变化时，管理员需要手动更新此序列号的值，主服务器比对主从的序列号不一致，会立刻进行主从同步操作，否则会等待刷新时间到了才由从服务器同步主服务器数据。|
| value4 | 刷新时间，表示从服务器从主服务器请求同步解析的时间间隔，默认以秒为单位，支持1h、1d表示，例如：2H。|
| value5 | 	重试时间，表示从服务器请求同步失败时，再次尝试同步的时间间隔，应该比同步间隔小 ，例如10m。|
| value6 | 过期时间，表示从服务器联系不到主服务器时，辅助DNS在多长时间内认为其缓存是有效的， 例如1W。|
| value7 |  否定答案的TTL值，表示不存在的记录缓存时长, 将不正确的域名缓存起来，直接返回结果给用户，不需要查询。|

除了以上元素，底部的内容为不同的DNS解析记录，此处给出腾讯云文档：
*  第一列：<a href="https://support.dnspod.cn/help-name/">腾讯云文档</a>
![image.png](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202304270211990.png)
* 第三列：<a href="https://support.dnspod.cn/help-type/">腾讯云文档</a>
![image.png](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202304270211162.png)
* 第四列：<a href="https://docs.dnspod.cn/dns/help-a/">腾讯云文档</a>

#### 4. 检查配置文件，启动bind9
```bash
# 检查配置文件
sudo named-checkconf
sudo named-checkzone baidu.com db.baidu.com

#重启bind9
sudo service bind9 restart
# 或 sudo rndc reload | systemctl restart named.service
```

#### 5.修改本机dns服务器地址，指向bind9
```bash
sudo vim /etc/resolv.conf
```
修改内容为：
```bash
# This file is managed by man:systemd-resolved(8). Do not edit.
#
# This is a dynamic resolv.conf file for connecting local clients to the
# internal DNS stub resolver of systemd-resolved. This file lists all
# configured search domains.
#
# Run "resolvectl status" to see details about the uplink DNS servers
# currently in use.
#
# Third party programs must not access this file directly, but only through the
# symlink at /etc/resolv.conf. To manage man:resolv.conf(5) in a different way,
# replace this symlink by a static file or a different symlink.
#
# See man:systemd-resolved.service(8) for details about the supported modes of
# operation for /etc/resolv.conf.

#nameserver 127.0.0.53
nameserver 127.0.0.1
options edns0 trust-ad
```

#### 6.本机测试dns服务是否部署成功
```bash
ping baidu.com # 或其上配置的其他解析记录，如 aaa.baidu.com bbb.baidu.com
#或 nslookup baidu.com
#或 dig baidu.com
```
如下，ip指向自己小盒子即为成功
![image.png](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img/202304270225747.png)

**记得还原/etc/resolv.conf**

#### 延申
* 除了bind，常用dns服务端还有 dnsmasq 和 unbound。其中dnsmasq主打轻量、配置简单
* 刚才部署的bind除了小盒子本机访问，还可以在第二部讲的 `named.conf.options` 中配置监听53端口，则其他电脑将dns服务器配置为小盒子，即可访问baidu.com -> 小盒子ip的效果