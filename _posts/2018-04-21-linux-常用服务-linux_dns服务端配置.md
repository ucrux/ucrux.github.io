---
author: ucrux
comments: true
date: 2018-04-21 22:23:00 +0000
layout: post
title: linux dns服务端配置
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- 常用服务
---

- 正解: 主机查询到IP的流程
- 反解: IP反解析到主机名的流程
- zone: 无论正解反解,每个domain的记录就是区域(zone)

*一台DNS服务器可以管理多个domain*

- 正解相关
  - SOA : (start of authority) 起始授权机构
  - NS  : nameserver,记录的是DNS服务器
  - A   : address, 记录的是IP

- 反解相关
  - 反解是由IP找主机名

<!-- more -->

*互联网名称与数字地址分配机构(ICANN)对计算机使用的IP地址进行管理*

> PTR : PoinTeR的缩写

- hint
  - 每部DNS都需要的正解zone: hint
  - 记录 '.' 的zone的类型,就是hint

- master/slave 
  - master: 从位于本机的文件中读取zone数据
  - slave : 从master上定期自动同步zone文件

> 序列号:master修改内容后,需要增加序列号,对于master slave同步来说,序列号非常重要


- 客户端的设定
  - 1. /etc/hosts         :早期hostname对应IP的文件
  - 2. /etc/resolv.conf   :记录DNS服务器
  - 3. /etc/nsswitch.conf :决定优先级

- nslookup
  - 1. A    IP address
  - 2. MX   mail exchange
  - 3. NS   DNS server 
  - 4. TXT  text info
  - 5. SOA  起始授权机构

- dig
  - dig -t mx 163.com qq.com google.com     #查询邮件交换类别的记录
  - dig -t a 163.com qq.com google.com 
  - dig -t ns 163.com qq.com google.com
  - dig -t txt 163.com qq.com google.com
  - dig -t soa 163.com qq.com google.com
  - dig -x 118.89.52.139                    #反解

DNS server Forwarding配置
===

> 官网 https://www.isc.org/

## 下载软件包
```shell
wget http://ftp.isc.org/isc/bind9/9.7.1/bind-9.7.1.tar.gz
yum install -y make gcc openssl-devel perl
```

## 添加用户组及用户
```shell
groupadd -g 501 bind
useradd -u 501 bind -g bind -d /opt/named -s /sbin/nologin
```

## 编译安装
```shell
tar xf bind-9.7.1.tar.gz
cd bind-9.7.1
./configure --prefix=/opt/bind-9.7.1 --enable-threads
make -j4 && make install
ln -sv /opt/bind-9.7.1 /opt/named
```

## 修改权限
```shell
mkdir -pv /var/named
mkdir -pv /var/run/named
chown -R bind:bind /opt/bind-9.7.1 
chown -R bind:bind /var/named
chown -R bind:bind /var/run/named
chmod 700 /opt/named/etc
```

## 生成named.root 文件
```shell
cd /var/named
dig > named.root 
#or 
wget http://ftp.internic.net/domain/named.root      #反正我用的是这种
```

## 生成rndc.conf 写入 named.conf 文件中
```shell
/opt/named/sbin/rndc-confgen -r /dev/urandom > /opt/named/etc/rndc.conf
tail -10 /opt/named/etc/rndc.conf \
| head -9 | sed s/#\ //g > /opt/named/etc/named.conf
```

## bind 路径与 chroot
- 配置文件       :   /opt/named/etc/named.conf 
- zone file目录 :   /var/named
- pid-file目录  :   /var/run/named

## cache-only and forwarding DNS
- cache-only服务器:  仅有 . 这个 zone file,只缓存查询结果
- Forwarding服务器:  指定一台上层DNS服务器作为forwarding目标

**不需要设置正反解zone file,只需要设置named.conf**

## named.conf
在此文件中添加
```shell
vi /opt/named/etc/named.conf

options {
    listen-on port 53 { any; };   #监听任意端口
    directory       "/var/named"; #zone file目录

    allow-query     { any; };     #允许任意客户端查询此DNS server
    recursion   yes;     #让此dns仅可进行forwarding查询,只会将查询交给上层DNS
    forward only;
    forwarders { 114.114.114.114; 8.8.8.8; 8.8.4.4; };  #上层DNS的地址
};
```

## 启动dns服务
```shell
/opt/named/sbin/named -c /opt/named/etc/named.conf -u bind
```
**如果修改了 zone file, 可以用rndc重新加载**
```shell
/opt/named/sbin/rndc reload
```

## 测试
```shell
 dig www.baidu.com @127.0.0.1       #使用本机DNS
```

DNS主从
===
## DNS正解
```shell
dig www.ucrux.com @8.8.8.8

...
www.ucrux.com.    3599    IN  CNAME   ucrux.com.
ucrux.com.        3599    IN  A   118.89.52.139
...


domain  TTL(time to live)   IN(关键字Internet) RRtype  RRdata
```

- TTL     :单位秒,当此记录被其他DNS服务器查询到后,保持在对方DNS服务器的缓存中多长时间
- RRtype  :资源记录类型 
- RRdata  :资源记录数据

>   hostname.   IN  A               ipv4 address
>   hostname.   IN  AAAA            ipv6 address
>   domain.     IN  NS              manager of this domain
>   domain.     IN  SOA             管理此域名的七个重要参数
>   domain.     IN  MX              数字  接收邮件的服务器主机名字
>   主机别名.     IN  CNAME           实际的主机名
>   domain.     IN  TXT             文本信息,多以spf的文本格式出现

- dig -t aaaa www.google.com
- dig -t a www.google.com
- dig -t ns google.com
- dig -t mx google.com
  - google.com.     600 IN  MX  50 alt4.aspmx.l.google.com.
  - google.com.     600 IN  MX  40 alt3.aspmx.l.google.com.
  - google.com.     600 IN  MX  20 alt1.aspmx.l.google.com.
  - google.com.     600 IN  MX  10 aspmx.l.google.com.
  - google.com.     600 IN  MX  30 alt2.aspmx.l.google.com.
  - 数字是优先级的,数字越小,优先级越高
- dig -t soa google.com
  - 管理域名的服务器管理信息
  - ns3.google.com. dns-admin.google.com. 153423654 900 900 1800 60
    - 1. Master DNS服务器的主机名
    - 2. 管理员的邮件地址,即: dns-admin@google.com 
    - 3. 序列号:zone file的版本号,越大越新,主从同步时,修改zone file之后要将序列号改大,通常"YYYYMMDDNU"
    - 4. 刷新频率:单位秒,定义slave向master同步的频率
    - 5. 失败重试时间:slave同步失败后,多长时间之后重试
    - 6. 失效时间:如果尝试时间一直失败,在这个时间之后,slave失效,不再提供有效服务
    - 7. 缓存时间(Minumum TTL):TTL的默认值

> Refresh >= Retry * 2
> Refresh + Retry < Expire
> Expire > Retry * 10
> Expire >= 7 Days

- CNAME: 主机别名
- TXT: 保存域名的附加信息
  - 如 SPF,用于登记某域名拥有的用来外发邮件的所有IP地址
- 反解: dig -x 173.194.219.27
  - 27.219.194.173.in-addr.arpa. 86400 IN   PTR ya-in-f27.1e100.net.
  - 主机名尽量使用完整的FQDN,然后加上 .
- DNS 环境规划
  - 1. named.conf       :主要配置文件
  - 2. one.com.zone     :正解区配置文件
  - 3. 192.168.122.zone :反解区配置文件
  - 4. named.root       :root(.)正解配置文件


避免这个报错:managed-keys-zone ./IN: loading from master file managed-keys.bind failed: file not found
```shell
touch /var/named/managed-keys.bind
```

### named.conf
#### mastar DNS
```shell
vi /opt/named/etc/named.conf

options {
    listen-on port 53 { any; };                         #监听任意端口
    directory       "/var/named";                       #zone file目录
    
    pid-file        "named.pid";

    allow-query     { any; };                           #允许任意客户端查询此DNS server
    allow-transfer  { none; };
    forwarders { 114.114.114.114; 8.8.8.8; 8.8.4.4; };  #上层DNS的地址
};

zone "." IN {
    type hint;                                          #类型 根
    file "named.root";
};

zone "one.com" IN {
    type master;
    file "one.com.zone";
    allow-update { none; };
    allow-transfer { 192.168.122.104; };
    notify yes;
    also-notify {192.168.122.104; } ;
};

zone "122.168.192.in-addr.arpa" IN {
    type master;
    file "192.168.122.zone";
    allow-transfer { 192.168.122.104; };
};
```
> type: 针对 . 的hint; 需要手动修改的master; 可自动更新的slave
> file: zone file 的文件名
> 反解zone: in-addr.arpa 

#### slave DNS
**slave DNS没有区文件,只需要配置好named.conf,去文件同步自master**
```shell
vi /opt/named/etc/named.conf

options {
    listen-on port 53 { any; };                         #监听任意端口
    directory       "/var/named";                       #zone file目录
    
    pid-file        "named.pid";

    allow-query     { any; };                           #允许任意客户端查询此DNS server
    allow-transfer  { none; };
    forwarders { 114.114.114.114; 8.8.8.8; 8.8.4.4; };  #上层DNS的地址
};

zone "." IN {
    type hint;                                          #类型 根
    file "named.root";
};

zone "one.com" IN {
    type slave;
    file "one.com.zone";
    masters { 192.168.122.103; };
};

zone "122.168.192.in-addr.arpa" IN {
    type slave;
    file "192.168.122.zone";
    masters { 192.168.122.103; };
};
```

### 正解文件(zone文件)

#### master DNS
```shell
vi /var/named/one.com.zone

$ttl 38400
one.com.    IN      SOA     ns1.one.com. dns-admin.one.com. (
                            2017041800
                            10800
                            3600
                            604800
                            38400
                            )
            IN      NS      ns1.one.com.
            IN      NS      ns2.one.com.
ns1         IN      A       192.168.122.103
ns2         IN      A       192.168.122.104
@           IN      MX  10  mail
www         IN      A       192.168.122.103
blog        IN      CNAME   www
bbs         IN      CNAME   www
ftp         IN      CNAME   www
mail        IN      A       192.168.122.104
```
### 反解文件(zone文件)
#### master DNS
```shell
vi /var/named/192.168.122.zone

$TTL 600
@           IN      SOA     ns1.one.com. dns-admin.one.com. (
                            2017041800
                            1D
                            1H
                            1W
                            3H
                            )
@           IN      NS      ns1.one.com.
@           IN      NS      ns2.one.com.
103         IN      PTR     ns1.one.com.
104         IN      PTR     ns2.one.com.
103         IN      PTR     www.one.com.
```

### 配置文件检查
```shell
/opt/named/sbin/named-checkconf /opt/named/etc/named.conf                               #检查配置文件
/opt/named/sbin/named-checkzone one.com /var/named/one.com.zone                         #检查正解区文件
/opt/named/sbin/named-checkzone 122.168.192.in-addr.arpa /var/named/192.168.122.zone    #检查反解区文件
```

### 启动服务
```shell
/opt/named/sbin/named -c /opt/named/etc/named.conf -u bind
```



DNS view
===

以某种特殊的方式,根据用户来源的不同,而返回不同的查询结果.

```shell
acl "cnc" {192.168.1.0/24;192.168.2.0/24};

view "internal" {
    match-clients { 192.168.122.10/24; } ;
};

view "internal2" {
    match-clients { "cnc"; } ;
};
```
> internal是区域名称,可以自定义,但必须唯一
> 区域区分是通过match-clients实现,internal只处理来自192.168.122.10/24网段的请求
> match-clients可引入acl,如internal2区域

## view 配置示例
### master dns
```shell
#named.conf
acl "cnc" {192.168.122.0/24;} ;

view "internal" {
    match-clients { "cnc"; } ;
    zone "one.com" {
        type master ;
        file "one.cone.cnc.zone";
    };
};

view "extenal" {
    match-clients { any; } ;
    recursion no;
    zone "one.com" {
        type master ;
        file "one.cone.any.zone";
    };
};
```

### slave DNS
```shell
# 视图对比是自上而下的,匹配到一个视图之后,就不会继续匹配了
# slave DNS 需要有与主服务器的视图数量相匹配的ip地址数量
# transfer-source 指定slave服务器的不同的IP地址,即通过slave上的哪个ip向slave传送数据
# 每个 view 要对应 slave 上的一个IP
# named.conf
acl "cnc" {192.168.122.0/24;} ;

view "internal" {
    match-clients { "cnc"; } ;
    transfer-source 192.168.122.104 ;
    zone "one.com" {
        type slave ;
        master {192.168.122.103;};
        file "one.cone.cnc.zone";
    };
};

view "extenal" {
    match-clients { any; } ;
    transfer-source 192.168.122.105 ;
    recursion no;
    zone "one.com" {
        type slave ;
        master {192.168.122.103;};
        file "one.cone.any.zone";
    };
};
```

## view配置规划
- 1. /var/named/dx/one.com.zone       电信正解文件  
- 2. /var/named/wt/one.com.zone       网通正解文件  
- 3. /var/named/ot/one.com.zone       其他正解文件  
- 4. /var/named/dx.cfg                匹配电信IP段文件
- 5. /var/named/wt.cfg                匹配网通IP段文件

** 注意:slave需要3个IP地址**
```shell
ifconfig eth0 192.168.122.104/24 up
ifconfig eth0:0 192.168.122.105/24 up 
ifconfig eth0:1 192.168.122.106/24 up 
```


```                       _______slave______       
            |-- view dx --| 192.168.122.104|
            |             |                |
            |             |                |
    master  |-- view wt --| 192.168.122.105| 
            |             |                |
            |             |                |
            |-- view ot --| 192.168.122.106| 
                   
```

### 具体配置过程
```shell
cd /var/named
mkdir dx wt ot
chown bind:bind dx wt ot
touch dx.cfg wt.cfg 
```

#### master named.conf 
```shell
include "dx.cfg" ;
include "wt.cfg" ;

view "wtzone" {
    match-clients { wt;192.168.122.104;!192.168.122.105;!192.168.122.106; } ;
    recursion yes;
    allow-update { none; } ;
    allow-transfer { 192.168.122.104; } ;
    notify yes ;
    also-notify { 192.168.122.104; } ;
    
    zone "." IN {
        type hint;
        file "named.root";
    };

    zone "one.com" {
        type master ;
        file "wt/one.com.zone";
    };
};


view "dxzone" {
    match-clients { dx;!192.168.122.104;192.168.122.105;!192.168.122.106; } ;
    recursion yes;
    allow-update { none; } ;
    allow-transfer { 192.168.122.105; } ;
    notify yes ;
    also-notify { 192.168.122.105; } ;
    
    zone "." IN {
        type hint;
        file "named.root";
    };

    zone "one.com" {
        type master ;
        file "dx/one.com.zone";
    };
};

view "otzone" {
    match-clients { any;!192.168.122.104;!192.168.122.105;192.168.122.106; } ;
    recursion yes;
    allow-update { none; } ;
    allow-transfer { 192.168.122.106; } ;
    notify yes ;
    also-notify { 192.168.122.106; } ;
    
    zone "." IN {
        type hint;
        file "named.root";
    };

    zone "one.com" {
        type master ;
        file "ot/one.com.zone";
    };
};
```

#### slave named.conf 
```shell
include "dx.cfg" ;
include "wt.cfg" ;

view "wtzone" {
    match-clients { wt;192.168.122.104;!192.168.122.105;!192.168.122.106; } ;
    transfer-source 192.168.122.104;
    allow-notify { 192.168.122.104; } ;
    recursion yes;
    
    zone "." IN {
        type hint;
        file "named.root";
    };

    zone "one.com" {
        type slave ;
        file "wt/one.com.zone";
        masters { 192.168.122.103; } ;
    };
};


view "dxzone" {
    match-clients { wt;!192.168.122.104;192.168.122.105;!192.168.122.106; } ;
    transfer-source 192.168.122.105;
    allow-notify { 192.168.122.105; } ;
    recursion yes;
    
    zone "." IN {
        type hint;
        file "named.root";
    };

    zone "one.com" {
        type slave ;
        file "dx/one.com.zone";
        masters { 192.168.122.103; } ;
    };
};


view "otzone" {
    match-clients { wt;!192.168.122.104;!192.168.122.105;192.168.122.106; } ;
    transfer-source 192.168.122.106;
    allow-notify { 192.168.122.106; } ;
    recursion yes;
    
    zone "." IN {
        type hint;
        file "named.root";
    };

    zone "one.com" {
        type slave ;
        file "ot/one.com.zone";
        masters { 192.168.122.103; } ;
    };
};
```


#### zone file only in master
##### dx/one.com.zone
```shell
$ttl 38400
one.com.    IN      SOA     ns1.one.com. dns-admin.one.com. (
                            2017041800
                            10800
                            3600
                            604800
                            38400
                            )
            IN      NS      ns1.one.com.
            IN      NS      ns2.one.com.
ns1         IN      A       192.168.122.103
ns2         IN      A       192.168.122.104
@           IN      MX  10  mail
www         IN      A       192.168.122.103
blog        IN      CNAME   www
bbs         IN      CNAME   www
ftp         IN      CNAME   www
mail        IN      A       192.168.122.104
```
##### wt/one.ome.zone
```shell
$ttl 38400
one.com.    IN      SOA     ns1.one.com. dns-admin.one.com. (
                            2017041800
                            10800
                            3600
                            604800
                            38400
                            )
            IN      NS      ns1.one.com.
            IN      NS      ns2.one.com.
ns1         IN      A       192.168.122.103
ns2         IN      A       192.168.122.104
@           IN      MX  10  mail
www         IN      A       10.0.0.103
blog        IN      CNAME   www
bbs         IN      CNAME   www
ftp         IN      CNAME   www
mail        IN      A       10.0.0.104
```
##### ot/one.ome.zone
```shell
$ttl 38400
one.com.    IN      SOA     ns1.one.com. dns-admin.one.com. (
                            2017041800
                            10800
                            3600
                            604800
                            38400
                            )
            IN      NS      ns1.one.com.
            IN      NS      ns2.one.com.
ns1         IN      A       192.168.122.103
ns2         IN      A       192.168.122.104
@           IN      MX  10  mail
www         IN      A       172.16.122.103
blog        IN      CNAME   www
bbs         IN      CNAME   www
ftp         IN      CNAME   www
mail        IN      A       172.16.122.104
```


#### dx.cfg and wt.cfg 
这两个文件 master 和 slave 都有
##### dx.cfg 
```
acl dx {192.168.122.101;} ;
```

##### wt.cfg
```
acl wt {192.168.122.102;} ;
```

DNS 日志
===
/var/log/message中不是专门的bind日志,详细日志可如下配置

add in named.conf after section "options" 
```
logging {
    channel <string>; {
        file <logfile> ;
        syslog <optional_facility> ;
        null;
        stderr;
        severity <logseverity> ;        #critical error warning notice info 
        print-time <boolean> ;
        print-severity <boolean> ;
        print-category <boolean> ;
    };

    category <string>;
        { <string>;...};
};
# 通道 channel  通过什么记录日志
# 类别 category 日志的类别
```



## 记录查询日志的配置
add in named.conf after section "options"
```
logging {
    channel query_log {
        file "query.log" versions 3 size 200m ;  ;versions 3,so query.log query.log0 query.log1 query.log2
        severity info ;
        print-category yes ;
        print-severity yes ;
        print-time yes ;
    } ;

    category queries {
        query_log ;
        ;default_syslog ;
        default_debug ;
    } ;
};
```


获取各运营商的IP地址
===

```shell
wget http://ftp.apnic.net/apnic/dbase/tools/ripe-dbase-client-v3.tar.gz
tar xzvf ripe-dbase-client-v3.tar.gz
cd whois-3.1
./configure
make
make install
```

## 获取网通IP地址库
```shell
./whois3 -h whois.apnic.net -l -i mb MAINT-CNCGROUP-RR > data/cnc-rr
```
### 格式化网通地址
```shell
cat data/cnc-rr|grep route|sed 's/route://g'|sed 's/. //g'
```

## 获取中国电信IP地址库
```shell
./whois3 -h whois.apnic.net -l -i mb MAINT-CHINANET > data/chinanet
```

## 获取中国铁通
```shell
./whois3 -h whois.apnic.net -l -i mb MAINT-CN-CRTC > data/crtc
```

