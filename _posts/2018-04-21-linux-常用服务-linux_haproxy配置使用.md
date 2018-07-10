---
author: asmbits
comments: true
date: 2018-04-21 22:41:00 +0000
layout: post
title: linux haproxy配置使用
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- 常用服务
---

>官网:http://haproxy.1wt.eu

- 1. 健康检查
- 2. 会话保持:在会话当中插入cokie
- 3. 四层,七层代理:TCP,HTTP 
- 4. frontend,backend

<!-- more -->

实验配置
===

>先决条件:已安装heartbeat

## 拓扑

| 主机名      | IP             | 用途 |
|-------------|-----------------|------|
| heartbeat01 | 192.168.122.101 |管理IP|
|             | 10.0.0.101      |心跳IP|
|             | 192.168.122.103 | VIP  |
| heartbeat02 | 192.168.122.102 |管理IP|
|             | 10.0.0.101      |心跳IP|


## 安装配置haproxy
```shell
#获取软件包
wget http://haproxy.1wt.eu/download/1.4/src/haproxy-1.4.24.tar.gz

tar xf haproxy-1.4.24.tar.gz
cd haproxy-1.4.24 
make TARGET=linux2628 ARCH=x86_64
make PREFIX=/opt/haproxy-1.4.24 install
```

**开启内核转发**
```shell
sed -i 's#net.ipv4.ip_forward = 0#net.ipv4.ip_forward = 1#g' /etc/sysctl.conf

sysctl -p
```
> 其实可以不打开转发,因为直接使用代理而不是NAT

### 创建haproxy所需要的目录
```shell
cd /opt/haproxy-1.4.24
mkdir -pv bin conf logs var/run var/chroot
```

### haproxy配置文件
#### 说明
```shell
global                                                  #全局配置参数段,控制启动前的进程及系统相关设置
    chroot /opt/haproxy-1.4.24/var/chroot               #安全作用
    daemon                                              #以守护进程方式运行
    user    haproxy                                     #以此用户运行
    group   haproxy                                     #以此组运行
    log     127.0.0.1:514   local0  info                #定义日志设备和日志级别
    pidfile /opt/haproxy-1.4.24/var/run/haproxy.pid     #进程号文件
    maxconn 20480                                       #最大连接数
                                                        #由于每个连接包括一个客户端和一个服务端
                                                        #实际最大连接数是该值的两倍
    spread-checks   3
    #tune.maxaccept 100
    #tune.maxpollevents 180
    nbproc  8                                           #haproxy启动时的进程数,该值最好不超过cpu逻辑核心数

defaults    #配置默认参数,如果frontend,backend,listen等段未设置,则使用defaults段配置
    log     global
    #log    127.0.0.1   local3  err
    mode    http                                        #mode {http|tcp|health},http:7层模式;tcp四层模式,health是健康检查
    #option httplog                                     #启用日志记录http请求
    option  dontlognull                                 #日志不记录空连接
    retries 3                                           #健康检查重试次数
    option  redispatch                                  #使用cookie时,会将后端服务器的serverid插入cookie中
    contimeout  5000                                    #毫秒,成功连接一台服务器的最长等待时间
    clitimeout  50000                                   #毫秒,连接客户端发送数据时的成功连接最长等待时间
    srvtimeout  50000                                   #毫秒,服务段回应客户端数据发送的最长等待时间
    #ulimit -n 65535                                    #最大打开文件描述符数量,建议不设置
    option  abortonclose                                #当负载较高时,自动结束但前队列处理比较久的连接
    option  httpclose                                   #完成一次传输后会主动关闭TCP连接

listen  www 10.0.0.7:80                                 #指定监听名,地址,端口. listen可以有多个,可以绑定不同的ip,端口,服务
    no  option  splice-response
    #以下为web界面
    stats   enable
    stats   uri /admin?status
    stats   auth    proxy:one                           #用户名:密码
    ########
    balance roundrobin                                  #负载均衡策略
    option  forwardfor                                  #把用户的ip转给real server
    #健康检查相关
    option  httpchk HEAD    /chech.html HTTP/1.0
    server  www01   10.0.0.9:80 check                   #real server
    server  www02   10.0.0.8:80 check                   #real server
    ########
frontend    #匹配接收客户所请求的域名,uri等.针对不同匹配,作不同处理
backend     #定义后端服务器集群,以及对后端服务器的一些权重,队列,连接数等选项的设置
```

#### 配置文件
```shell
#添加haproxy用户和组
useradd haproxy -m -s /sbin/nologin

#vi /opt/haproxy-1.4.24/conf/haproxy.conf
global
    chroot /opt/haproxy-1.4.24/var/chroot
    daemon
    user    haproxy
    group   haproxy
    log     127.0.0.1:514   local0  warning
    pidfile /opt/haproxy-1.4.24/var/run/haproxy.pid
    maxconn 20480
    spread-checks   3
    nbproc  8

defaults
    log     global
    mode    http
    retries 3
    option  redispatch
    contimeout  5000
    clitimeout  50000
    srvtimeout  50000
    option  abortonclose
    option  httpclose

listen  one
    bind    192.168.122.103:80
    mode    http
    stats   enable
    stats   hide-version
    stats   uri /admin?status           # http://192.168.122.103/admin?status 可以查看RS的状态 
    stats   auth    admin:one           # 这是以上页面的用户名和密码
    balance roundrobin
    option  forwardfor                  #必须放在listen下面,是web RS可以记录真实的用户IP,而不是haproxy的IP
    cookie SERVERID insert indirect
    timeout server 15s
    timeout connect 15s
    server  www01   192.168.122.101:80  cookie A check port 80 inter 50000 fall 5       #check port 后面是健康检查的端口
    server  www02   192.168.122.188:80  cookie A check port 80 inter 5000 fall 5
```

> 如果希望RS记录用户真实IP,那么RS的日志格式也需要修改(以apache为例)<br>
> LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined<br>
>                                   |<br>
>                                   V<br>
> LogFormat "\"{X-Forwarded-For}i\" %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined

### 启动服务
```shell
#检查配置文件
/opt/haproxy-1.4.24/sbin/haproxy -c -f /opt/haproxy-1.4.24/conf/haproxy.conf

#启动服务
/opt/haproxy-1.4.24/sbin/haproxy -f /opt/haproxy-1.4.24/conf/haproxy.conf  -D

-D goes daemon
-q quiet mode: dont display message
-c check mode: only check config files and exit
-n sets the maximum total # of connections (2000)
-m limits the usable amount of memory
-N sets the defaults, per-proxy maximum # of connections (2000)
-p wirtes pids of all children to this file 
-de disables epoll() usage even when available
-ds disables speculative epoll() usage even when available
-dp disables poll() usage even when available
-sf/-st [pid]* finishes/terminates old pids, must be last argument #用于平滑重启
```

### haproxy日志配置
```shell
cat >> /etc/rsyslog.conf << EOF
# haproxy
local0.*    /opt/haproxy-1.4.24/logs/haproxy.log 
# end haproxy
EOF


vi /etc/sysconfig/rsyslog
#change
SYSLOGD_OPTIONS="-c 5"
#to 
SYSLOGD_OPTIONS="-c 2 -m 0 -r -x"

# -m 0: disable 'MARK' messages
# -r enable logging from remote machines
# -x disable DNS lookups on messages recieved with -r


#修改配置文件,使其支持远程日志
#取消注释
vi +13 /etc/rsyslog.conf
$ModLoad imudp
$UDPServerRun 514

#重启rsyslog
/etc/init.d/rsyslog restart
```

## haproxy健康检查
### 基于tcp端口的健康检查
```shell
server www21 10.0.0.101:80  check port 80
server www22 10.0.0.101:80  check                   #默认检查冒号后面的端口
server www23 10.0.0.101:80  check port 80 inter 5000 fall 5
```
>inter 5000 fall 5 : 每5秒检查一次,共检查5次<br>
>如果不加 inter fall,默认每2秒检查一次,共检查3次<br>

**相关参数缺省值**
> inter 2000<br>
> rise 2 :在RS宕机后,检查2次ok,将其重新加入负载<br>
> fall 3<br>
> port : default server port<br>
> addr : specific address for the test (default= address server)<br>

### 基于http的ip URL的检查方式
#### 利用head
```shell
option httpchk HEAD /check.html HTTP/1.0
#这种检查方法相当于 curl -I http://10.0.0.8/check.html

server www21 10.0.0.101:80 maxconn 2048 weight 8 check inter 3000 fall 2 rise 2
```
> maxconn 控制此节点的最大并发<br>
> weight 权重大的被分配的概率越大<br>

#### 利用GET
```shell
option httpchk GET /index.html

server www01 10.0.0.101:80 maxconn 2048 weight 8 check inter 3000 fall 2 rise 2
```

#### 利用URL
```shell
option httpchk HEAD /index.jsp HTTP/1.1\r\nHost:\ www.asmbits.com
option httpchk GET /index.jsp HTTP/1.1\r\nHost:\ www.asmbits.com
```
> 使用域名检查的话, RS的hosts文件要指定解析地址才有意义<br>
> 10.0.0.101 www.asmbits.com<br>


### 其他服务的健康检查
```
option ldap-check 
    Use LDAPv3 health checks for server testing

option mysql-check [ user <username> ]
    Use MySQL health checks for server testing

option smtpchk
option smtpchk <hello> <domain>
    Use SMTP health chechs for server testing

```

- backup参数
  - 当www09和www10全挂了,backup节点才会启用
  - option allbackups           #如果不加这个参数,只会启动一台backup机器
  - server www09 10.0.0.101:80 check 
  - server www10 10.0.0.102:80 check 
  - server www11 10.0.0.103:80 check backup 
  - server www12 10.0.0.104:80 check backup 

### haproxy服务启动脚本

> haproxyd

```shell
#!/bin/sh 

HAPROXY_BIN="/opt/haproxy-1.4.24/sbin/haproxy"
HAPROXY_CONF_FILE="/opt/haproxy-1.4.24/conf/haproxy.conf"

function usage()
{
    echo "$(basename $0) start|stop"
}

function haproxy_start()
{
    ${HAPROXY_BIN} -f ${HAPROXY_CONF_FILE} -D
}

function haproxy_stop()
{
    pkill haproxy
}

case $1 in 
    start)
        haproxy_start ;;
    stop)
        haproxy_stop ;;
    *)
        usage ;;
esac
```

## haproxy自身高可用

1. 将上面的脚本拷贝到/etc/ha.d/resource.d/中,用于heartbeat资源管理
2. 在原有/etc/ha.d/haresource中配置

```shell
heartbeat01 IPaddr::192.168.122.103/24/eth0 drbddisk::data Filesystem::/dev/drbd0::/md1::ext4 rsmd1 haproxyd
```


**解决haproxy配置文件绑定的IP,在本地网卡上并不存在**
```shell
echo "net.ipv4.ip_nonlocal_bind = 1" >> /etc/sysctl.conf
```


## frontend,backend
> frontend,匹配规则<br>
> backend,RS服务器的地址资源池<br>

```
#在defaults中添加
defaults
    stats   enable
    stats   hide-version
    stats   uri /admin?status
    stats   auth    admin:one

#删除listen,并添加如下
frontend asmbitsone
    bind 192.168.122.103:80
    acl one_dom hdr(host) -i www.one.com  #one_dom规则名,hdr(host)函数表示去地址里的域名,-i不区分大小写
    acl bbs_dom hdr(host) -i bbs.asmbits.com 
    acl www_dom hdr(host) -i www.asmbits.com 
    acl tmp_dom hdr(host) -i tmp.asmbits.com 

    redirect prefix http://www.baidu.com code 301 if one_dom    #如果符合one_dom的规则就以301的形式跳转
    use_backend bbsasmbits if bbs_dom                           #如果符合bbs_dom的规则就找bbsasmbits这个backend
    use_backend wwwasmbits if www_dom
    default_backend wwwasmbits                                  #缺省backend,例如tmp_dom就会走这条规则
    

backend wwwasmbits
    balance leastconn                   #最小连接数负载算法
    option httpclose
    option forwardfor
    #cookie SERVERID insert indirect
    server www01 192.168.122.101:8001 maxconn 10240 weight 50 check inter 3000 fall 3
    server www01 192.168.122.102:8001 maxconn 10240 weight 50 check inter 3000 fall 3

backend bbsasmbits
    balance leastconn                   #最小连接数负载算法
    option httpclose
    option forwardfor
    #cookie SERVERID insert indirect
    server bbs01 192.168.122.101:8002 maxconn 10240 weight 50 check inter 3000 fall 3
    server bbs01 192.168.122.102:8002 maxconn 10240 weight 50 check inter 3000 fall 3
```


- acl <aclname> <criterion> [flags] [operator] <value> ...
  - Declare or complete an access list.
  - May be used in sections :   defaults | frontend | listen | backend
  -                                no    |    yes   |   yes  |   yes
    - Example:
      - acl invalid_src  src          0.0.0.0/7 224.0.0.0/3
      - acl invalid_src  src_port     0:1023
      - acl local_dst    hdr(host) -i localhost

### 根据目录进行过滤转发
```
acl asmbits_static  path_beg    /static/
acl asmbits_php     path_beg    /php/
acl asmbits_java    path_beg    /resin/

use_backend static_pool if  asmbits_static
use_backend php_pool    if  asmbits_php
use_backend java_pool   if  asmbits_java
```

### 基于扩展名的跳转
```
acl asmbits_static  path_end    .gif .png .jpg .css .js
acl asmbits_pic     url_reg     \.(css|js|html|png|css?.*|js?.*)$  #使用正则表达式

use_backend static_pool if  asmbits_static or asmbits_pic
```

### 基于user_agent的7层跳转

> iphone 客户端访问 3g-iphone<br>
> android 客户端访问 3g-android<br>

```
acl iphone_users    hdr_sub(user-agent) -i iphone 
redirect    prefix  http://3g-iphone.asmbits.com    if  iphone_users


acl android_users   hdr_sub(user-agent) -i android 
redirect    prefix  http://3g-android.asmbits.com   if  android_users
```
### 基于IP和端口控制过滤
```
acl valid_ip    src 192.168.122.0/24
block if !valid_ip                          #不符合 valid_ip 直接拒绝访问
```

## 错误代码跳转
**不支持404**
```
errorfile <code> <file>
  Return a file contents instead of errors generated by HAProxy
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  Arguments :
    <code>    is the HTTP status code. Currently, HAProxy is capable of
              generating codes 200, 400, 403, 408, 500, 502, 503, and 504.

    <file>    designates a file containing the full HTTP response. It is
              recommended to follow the common practice of appending ".http" to
              the filename so that people do not confuse the response with HTML
              error pages, and to use absolute paths, since files are read
              before any chroot is performed.


errorrloc <code> <url>
errorloc302 <code> <url>
  Return an HTTP redirection to a URL instead of errors generated by HAProxy
  May be used in sections :   defaults | frontend | listen | backend
                                 yes   |    yes   |   yes  |   yes
  Arguments :
    <code>    is the HTTP status code. Currently, HAProxy is capable of
              generating codes 200, 400, 403, 408, 500, 502, 503, and 504.

    <url>     it is the exact contents of the "Location" header. It may contain
              either a relative URI to an error page hosted on the same site,
              or an absolute URI designating an error page on another site.
              Special care should be given to relative URIs to avoid redirect
              loops if the URI itself may generate the same error (eg: 500).
```

