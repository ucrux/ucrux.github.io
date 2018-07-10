---
author: ucrux
comments: true
date: 2018-04-21 22:47:00 +0000
layout: post
title: linux openvpn配置使用
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- 常用服务
---

- openVPN 使用openSSL加密,使用SSLv3/TLSv1协议

openvpn server部署
===
## 操作系统环境
```shell
cat /etc/redhat-release 
CentOS release 6.8 (Final)

uname -r
2.6.32-642.13.1.el6.x86_64
```

<!-- more -->

## 安装前准备
### 时间同步
```shell
yum install ntpdate -y
ntpdate pool.ntp.org

echo "#time sync" >> /var/spool/cron/root
echo "*/5 * * * * /usr/sbin/ntpdate pool.ntp.org" >> /var/spool/cron/root
```

## 安装openvpn
```shell
cd /software

#数据压缩模块
wget http://www.oberhumer.com/opensource/lzo/download/lzo-2.06.tar.gz
tar zxf lzo-2.06.tar.gz
cd lzo-2.06
./configure --prefix=/opt/sharelib/lzo-2.06
make -j4
make install

#安装openvpn
yum install -y openssl-devel
cd /software
wget https://swupdate.openvpn.org/community/releases/openvpn-2.2.2.tar.gz
tar zxf openvpn-2.2.2.tar.gz
cd openvpn-2.2.2

./configure --prefix=/opt/openvpn-2.2.2 --with-lzo-headers=/opt/sharelib/lzo/include \
--with-lzo-lib=/opt/sharelib/lzo/lib

make -j4
make install
```

## 配置openvpn server
### 建立CA根证书
```shell
cd /software/openvpn-2.2.2/easy-rsa/2.0

cp vars vars.bak

vi +64 vars
#change
export KEY_COUNTRY="CN"
export KEY_PROVINCE="Guangdong"
export KEY_CITY="Shenzhen"
export KEY_ORG="ucrux"
export KEY_EMAIL="one@ucrux.com"
export KEY_EMAIL=one@ucrux.com
export KEY_CN=ucrux
export KEY_NAME=ucrux
export KEY_OU=ucrux
export PKCS11_MODULE_PATH=changeme
export PKCS11_PIN=1234

source vars
#NOTE: If you run ./clean-all, I will be doing a rm -rf on /software/openvpn-2.2.2/easy-rsa/2.0/keys
./clean-all
./build-ca

ll keys/
total 12
-rw-r--r-- 1 root root 1330 Feb  5 18:27 ca.crt       #公钥证书
-rw------- 1 root root  916 Feb  5 18:27 ca.key       #私钥证书
-rw-r--r-- 1 root root    0 Feb  5 18:27 index.txt
-rw-r--r-- 1 root root    3 Feb  5 18:27 serial
```

### 生成服务端的密钥文件
```
./build-key-server ucruxSrv

ll keys/
total 40
...
-rw-r--r-- 1 root root 4037 Feb  5 18:32 ucruxSrv.crt
-rw-r--r-- 1 root root  716 Feb  5 18:32 ucruxSrv.csr
-rw------- 1 root root  916 Feb  5 18:32 ucruxSrv.key
...
```

### 生成客户端证书
```
./build-key client1

ll keys/
total 64
...
-rw-r--r-- 1 root root 3913 Feb  5 18:35 client1.crt
-rw-r--r-- 1 root root  712 Feb  5 18:35 client1.csr
-rw------- 1 root root  912 Feb  5 18:35 client1.key
...

#带密码的客户端证书
./build-key-pass client2
```

### generate diffle hellman parameters
生成传输进行密钥交换时用到的交换密钥协议文件
```
./build-dh

ll keys/
total 84
...
-rw-r--r-- 1 root root  245 Feb  5 19:04 dh1024.pem
...

```


|    filename    |      needed by       |       purpose       | secret |
|----------------|----------------------|---------------------|--------|
| ca.crt         | server + all clients | root CA certificate | NO     |
| ca.key         | signing machine only | root CA key         | YES    |
| dh{n}.pem      | server only          | diffie hellman para | NO     |
| ucruxSrv.crt | server only          | server certificate  | NO     |
| ucruxSrv.key | server only          | server key          | YES    |
| client{n}.crt  | client only          | client certificate  | NO     |
| client{n}.key  | clinet only          | client key          | YES    |


### HMAC firewall
防止而已攻击(DDOS,UDP port flooding)
```
openvpn --genkey --secret keys/ta.key

ll keys/ta.key 
-rw------- 1 root root 636 Feb  5 19:15 keys/ta.key
```

### 拷贝keys及配置
```
mkdir -pv /etc/openvpn
cd /software/openvpn-2.2.2/easy-rsa/2.0/
cp -ap keys /etc/openvpn/

cd /software/openvpn-2.2.2/sample-config-files/
cp client.conf server.conf /etc/openvpn/

cd /etc/openvpn/
cp server.conf server.conf.bak

grep -vE ";|^#|^$" server.conf
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key  # This file should be kept secret
dh dh1024.pem
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
keepalive 10 120
comp-lzo
persist-key
persist-tun
status openvpn-status.log
verb 3
```


|               配置参数              |               参数说明               |
|-------------------------------------|--------------------------------------|
| local 10.0.0.8                      | 监听的IP地址                         |
| port 52115                          | 监听端口                             |
| porto udp                           | 监听协议,推荐tcp                     |
| dev tun                             | 路由模式,tap或者tun                  |
| ca ca.crt                           | ca证书,最好用绝对路径                |
| cert ucruxSrv.crt                 |                                      |
| key ucruxSrv.key                  |                                      |
| dh dh1024.pem                       |                                      |
| server 10.8.0.0 255.255.255.0       | 分配给client的地址池                 |
| ifconfig-pool-persist ipp.txt       |                                      |
| push "route 10.0.0.0 255.255.255.0" | 推到客户端的路由(可多次push)         |
| client-to-client                    | client 之间是否允许通信              |
| duplicate-cn                        | 允许多个客户端使用同一个账号         |
| keepalive 10 120                    | 10秒检查一次,120没回应即客户端已断线 |
| comp-lzo                            | 开启压缩功能                         |
| persist-key                         | 超时后保持上一次key                  |
| persist-tun                         | 保持上一次tun或tap                   |
| status openvpn-status.log           | 日志状态信息                         |
| log /var/log/openvpn.log            | 日志文件                             |
| verb 3                              | 指定日志文件冗余                     |


```
##server.conf
local 0.0.0.0
port 52115
proto tcp

dev tun

ca /etc/openvpn/keys/ca.crt
cert /etc/openvpn/keys/ucruxSrv.crt
key /etc/openvpn/keys/ucruxSrv.key
dh /etc/openvpn/keys/dh1024.pem
tls-auth /etc/openvpn/keys/ta.key 0

server 10.8.0.0 255.255.255.0

mode server
topology subnet
push "topology subnet"
ifconfig 10.8.0.1 255.255.0.0
push "route-gateway 10.8.0.1"

ifconfig-pool-persist ipp.txt

#客户端个性化定制化目录,例如固定其VPN ip
client-config-dir /etc/openvpn/ccd
###################################################
# ll /etc/openvpn/ccd/                            #
# total 8                                         #
# -rw-r--r-- 1 root root 38 Feb 10 10:27 client1  #
# -rw-r--r-- 1 root root 38 Feb 10 10:27 client2  #
#                                                 #
# cat /etc/openvpn/ccd/client1                    #
# ifconfig-push 10.8.0.10 255.255.255.0           #
###################################################

keepalive 10 120
comp-lzo
persist-key
persist-tun
status openvpn-status.log

client-to-client
duplicate-cn
log /var/log/openvpn.log
verb 3




##client.conf
client
dev tun
proto tcp
remote ucrux.com 52115
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert client1.crt
key client1.key
ns-cert-type server
tls-auth ta.key 1
comp-lzo
verb 3
```


### 调整内核参数
```
vi /etc/sysctl.conf
net.ipv4.ip_forward = 1

sysctl -p
```

### 启动openvpn server
```
nohup /usr/local/sbin/openvpn --config /etc/openvpn/server.conf &

#开机自启动
echo "#startup openvpn server" >> /etc/rc.local
echo "nohup /usr/local/sbin/openvpn --config /etc/openvpn/server.conf &" >> /etc/rc.local
```


VPNserver的其他内网机器的访问
===
## 为VPN网段添加网络路由(其他内网机器)
### 命令添加
```shell
route add -net 10.8.0.0/24 gw 172.16.1.28

#10.8.0.0/24 为 VPN 网段
#172.16.1.28 为 VPN server
```
### 修改文件添加
```shell
#method 1
vi /etc/sysconfig/network-script/route-eth0
#add
10.8.0.0.24 via 172.16.1.28

#method 2
vi /etc/sysconfig/static-routes
#add
any net 10.8.0.0/24 gw 172.16.1.28

#method 3
vi /etc/rc.local
#add
route add -net 10.8.0.0/24 gw 172.16.1.28
```
##或者在VPNserver上做NAT
```shell
/sbin/iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j SNAT --to-source 172.16.1.28
```

openvpn吊销用户证书
===
```shell

cd /software/openvpn-2.2.2/easy-rsa/2.0
source vars
./revoke-full 证书名
#可能需要注释openssl-x.x.x.cnf最后六行
#[ pkcs11_section ]
#engine_id = pkcs11
#dynamic_path = /usr/lib/engines/engine_pkcs11.so
#MODULE_PATH = $ENV::PKCS11_MODULE_PATH
#PIN = $ENV::PKCS11_PIN
#init = 0


#在 keys/crl.pem 会生产证书吊销的内容
cat keys/crl.pem

#查看证书情况
cat keys/index.txt

#在openvpn server的配置文件中增加吊销配置
cp keys/crl.pem /etc/openvpn/keys/
vi /etc/openvpn/server.conf
#add
crl-verify /etc/openvpn/keys/crl.epm
```

查看openvpn在线用户
===
```
#server.conf中需要添加
status openvpn-status.log
#每分钟刷新用户列表

cat /etc/openvpn/openvpn-status.log 
OpenVPN CLIENT LIST
Updated,Sat Mar 11 14:06:31 2017
Common Name,Real Address,Bytes Received,Bytes Sent,Connected Since
client2,113.87.212.70:50548,635133,511910,Fri Mar 10 19:30:27 2017
ROUTING TABLE
Virtual Address,Common Name,Real Address,Last Ref
10.8.0.11,client2,113.87.212.70:50548,Fri Mar 10 19:30:28 2017
GLOBAL STATS
Max bcast/mcast queue length,7
END
```


linux下openvpn client
===
```shell
#数据压缩模块
wget http://www.oberhumer.com/opensource/lzo/download/lzo-2.06.tar.gz
tar zxf lzo-2.06.tar.gz
cd lzo-2.06
./configure
make -j4
make install

#安装openvpn
yum install -y openssl-devel
cd /software
wget https://swupdate.openvpn.org/community/releases/openvpn-2.2.2.tar.gz
tar zxf openvpn-2.2.2.tar.gz
cd openvpn-2.2.2

./configure --with-lzo-headers=/usr/local/include \
--with-lzo-lib=/usr/local/lib

make -j4
make install

#配置
mkdir /etc/openvpn
vi /etc/openvpn/client.conf

client
dev tun
proto tcp
remote ucrux.com 52115
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert client1.crt
key client1.key
ns-cert-type server
tls-auth ta.key 1
comp-lzo
verb 3

#启动服务
/usr/local/sbin/openvpn --config /etc/openvpn/client.conf &
```

VPN配置翻墙
===
## 在server端配置文件中添加
```
push "redirect-gateway def1 bypass-dhcp bypass-dns"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DSN 8.8.4.4"
```
## 打开内核转发
```shell
sed -i 's#net.ipv4.ip_forward = 0#net.ipv4.ip_forward = 1#' /etc/sysctl.conf
```

## 开启放火墙NAT映射
```
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j SNAT --to-source 10.0.0.28
```

VPN集群和高可用
==
## 方案一

- 1. 使用同一套证书在多个openvpn上
- 2. 客户端通过配置文件连接不同的vpnserver
   客户端配置文件里面写多个 VPNserver

```
remote server1.mydomain
remote server2.mydomain
remote server3.mydomain

remote-random           #随机选一个server进行连接
resolv-retry 10         #60秒后重连
```

## 方案二
- 使用DNS轮询来负载均衡
- 客户端使用域名来访问vpnserver

>需要特别注意DNS缓存


VPN统一身份认真方案
===

## 文件认证方式

```shell
cd /etc/openvpn
cp server.conf server.conf.ori

vi server.conf
#add
#允许用户自定义脚本执行,否则认证脚本会无法使用
script-security 3

#通过用户名密码验证(checkpw.sh 返回0成功,1失败)
auth-user-pass-verify /etc/openvpn/checkpw.sh via-env

#客户端不需要证书,使用用户名密码验证
client-cert-not-required

#使用客户端的username作为common name
username-as-common-name
```


vi /etc/openvpn/checkpw.sh
```shell
#!/bin/bash 

#定义文件位置及时间戳
##用户密码文件
PASSFILE="/etc/openvpn/psw-file"
##登陆日志文件
LOG_FILE="/var/log/openvpn-password.log"
TIME_STAMP=$(date "+%Y-%m-%d %T")


#如果密码文件不可读
if [ ! -r "${PASSFILE}" ]
then
    echo "${TIME_STAMP}: could not open password file \"${PASSFILE}\" for reading." >> ${LOG_FILE}
    exit 1
fi

#根据用户名处理密码文件
CORRECT_PASSWORD=$(awk '!/^;/&&!/^#/&&$1=="'${username}'" {print $2;exit}' ${PASSFILE})

#如果密码为空
if [ "${CORRECT_PASSWORD}" = "" ]
then
    echo "${TIME_STAMP}: User does not exist: username=\"${username}\",password=\"${password}\"." >> ${LOG_FILE}
    exit 1
fi

#如果秘密匹配
if [ "${password}" = "${CORRECT_PASSWORD}" ]
then
    echo "${TIME_STAMP}: Successful authentication: username=\"${username}\",password=\"${password}\"." >> ${LOG_FILE}
    exit 0
fi

echo "${TIME_STAMP}: incorrect password: username=\"${username}\",password=\"${password}\"." >> ${LOG_FILE}
exit 1
```

## 创建密码文件
```shell
cat >> /etc/openvpn/psw-file << EOF
user1 123456
user2 456789
EOF

chmod 400 /etc/openvpn/psw-file

chattr +i /etc/openvpn/psw-file
```

## 客户端配置文件
```shell
client
dev tun
proto tcp
remote ucrux.com 52115
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
#key client1.key
#ns-cert-type server
tls-auth ta.key 1
comp-lzo
verb 3
auth-user-pass
```


openvpn通过LDAP认证
===

## openvpn server 安装 ldap 客户端

```shell
yum -y install openldap-clients openldap nscd nss-pam-ldapd nss_ldap python-ldap
```

## 通过LDAP认证所需的文件
### 用户授权文件
authfile.conf
```
#在这个文件中的用户才有权限登陆vpn
oldgirl
```

### ldap用户认证脚本
check_credit.py 
```python
#!/usr/bin/python
import sys
import os
import logging
import ldap

#setting for ldap
ldap_url = "ldap://192.168.122.10:389"
ldap_starttls = True  #in myldap if True,connect will failed, so i set it as False
ldap_dn = "uid=%s,ou=People,dc=one,dc=com"

#setting for logging
log_filename = "tmp.log"
log_format = "%(asctime)s %(levelname)s %(message)s"
log_level = logging.DEBUG

#setting for authorization
auth_filename = "/opt/users-allowed.conf"

def get_users(fpath):
    fp = open( fpath, 'rb' )
    lines = fp.readlines()
    fp.close()

    users = {}

    for line in lines :
        line = line.strip()
        if len(line) <= 0 or line.startswith('#') :
            continue
        users[line] = True
    return users

def get_credits( fpath ) :
    fp = open( fpath, "rb" )
    lines = fp.readlines()
    fp.close

    assert len(lines) >= 2, "invalid credit file"

    username = lines[0].strip()
    password = lines[1].strip()

    return (username,password)

def check_credits( username, password ) :
    passed = False
    ldap.set_option( ldap.OPT_PROTOCOL_VERSION, ldap.VERSION3 )
    l = ldap.initialize(ldap_url)

    if ldap_starttls :
        l.start_tls_s( )

    try:
        l.simple_bind_s(ldap_dn % (username,),password)
        passed = True
    except ldap.INVALID_CREDENTIALS, e:
        logging.error( "username,'%s'/password,'%s' failed verifying" %(username,password) )
    l.unbind()
    return passed

def main(argv):
    credit_fpath = argv[1]
    (username,password) = get_credits(credit_fpath)
    
    if len(username) <= 0 or len(password) <= 0 :
        logging.error("invalid credits for user '%s'" %username)
        return 1
    logging.info("user '%s',password '%s' request login" %(username,password))

    if check_credits( username, password ) :
        users = get_users(auth_filename)
        if not username in users :
            logging.error("user '%s' not authorized to access" %username)
            return 1
        logging.info("access of user '%s' granted" % username)
        return 0
    else :
        logging.error("access of user '%s' denied" % username)
        return 1

if __name__ == "__main__" :
    logging.basicConfig( format=log_format, filename=log_filename, level=log_level )
    if len(sys.argv) != 2 :
        logging.fatal( "usage: %s <credit-file>" % sys.argv[0] )
        sys.exit(1)

    rcode = 1
    try:
        rcode = main(sys.argv)
    except Exception, e:
        logging.fatal( "exception happened: %s" % str(e) ) ;
        rcode = 1
    
    sys.exit(rcode)
```

### 用于测试的用户密码文件

user.conf
```
oldgirl
rage1984

#测试命令
#python check_credit.py user.conf
#测试结果
#2017-03-29 21:02:52,205 INFO user 'oldgirl',password 'rage1984' request login
#2017-03-29 21:02:52,208 INFO access of user 'oldgirl' granted

```

## 配置openvpn LDAP 认证登陆
```shell
cp check_credit.py /etc/openvpn/

cd /etc/openvpn
vi server.conf
#注释
#for local file auth
#auth-user-pass-verify /etc/openvpn/checkpw.sh via-env
#client-cert-not-required
#username-as-common-name

#添加
auth-user-pass-verify /etc/openvpn/check_credit.py via-file
client-cert-not-required
username-as-common-name
```
