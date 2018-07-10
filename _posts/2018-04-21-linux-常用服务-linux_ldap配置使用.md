---
author: ucrux
comments: true
date: 2018-04-21 22:45:00 +0000
layout: post
title: linux ldap配置使用
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- 常用服务
---

- LDAP是微软活动目录(AD)的一个开源实现
- DNS是一个典型的大范围分布式目录服务的例子

<!-- more -->

最初LDAP是这样的
```
             LDAP         DAP
LDAP client ------> LDAP -------> X.500 server
                   gateway
```

常用术语说明

| 关键字 |      英文全称      |                     含义                    |
|--------|--------------------|---------------------------------------------|
| DC     | domain component   | 如: example.com dc=example,dc=com           |
| uid    | user id            | 用户ID                                      |
| ou     | organiztion Unit   | 组织单位                                    |
| cn     | common name        | 公共名称                                    |
| sn     | surname            | 姓                                          |
| dn     | Distinguished name | 唯一辨别名, uid=tom,ou=market,cd=one,dc=com |
| rdn    | relative dn        | 相对辨别名，"uid=tom"or"cn=thomas jison"    |
| c      | country            | 国家 CN or US                               |
| o      | organization       | 组织 "one, Inc"                             |

LDAP中数据存储在DIT(Directory Interchange Tree)中
LDAP目录用OU把数据从逻辑把数据分开(OU:容器条目)

- DN的两种设置
  - 1. 基于cn
  - 2. 基于uid

**LDIF格式用于LDAP数据导入导出**

例:
```
dn: uid=user01,ou=people,dc=one,dc=com
dn: cn=fucker,ou=perple,dc=one,dc=com
```

信息在目录中的组织
```
              root(baseDN)
                  /   \
                C=CN  C=US
                        |
                   ST=California
                        |
                     O=Acme
                    /      \
             OU=sales       OU=Marketing
                |
           cn=wangleizhi



                       root(BaseDN)       
                      /      |     \
                dc=net     dc=com   dc=cn
                             |
                           dc=one
                          /       \
                 ou=People         ou=Servers
                    |
                 uid=babs
```

- LDIF文件格式的特点
  - openldat2.3的LDIF格式
    - 1. 通过空行来分割条目或定义
    - 2. 以'#'开始的行为注释
    - 3. 属性:值.如: dn:dc=one,dc=com
    - 4. 属性可以重复赋值,objectclass就可以有多个(每条LDAP必须有一条objectclass属性)
    - 5. 每行的结尾不允许有空格
- LDAP的配置模式
  - 1. 基于目录的查询服务
  - 2. 目录查询代理
  - 3. 异机复制数据

LDAP主从同步
```
                            |        |
       |-1.update request-> | slave  | <--7.--- slurpd
       |<--- 2.referal ---- |        |             ^
       |                                           |
client |                                           6
       |                                           |
       |--3.New request---> |        |             |
       |<--- 4.response --- | master | ---5.--- replication
                            |        |            Log
```

LDAP服务的安装
===
## 安装前准备
```shell
#时间同步
crontab -e
*/20 * * * * ntpdate time.windows.com

#配置host
echo "10.0.0.31 one.com" >> /etc/hosts

#安装openldap
yum install -y openldap openldap-*
yum install -y nscd nss-pam-ldapd nss-* pcre pcre-*
```

## 配置ldap master
```shell
cd /etc/openldap/

#openldap2.4配置文件
ll /etc/openldap/slapd.d/cn\=config
total 80
drwx------ 2 ldap ldap  4096 Mar 13 22:47 cn=schema
-rw------- 1 ldap ldap 59366 Mar 13 22:47 cn=schema.ldif
-rw------- 1 ldap ldap   663 Mar 13 22:47 olcDatabase={0}config.ldif
-rw------- 1 ldap ldap   596 Mar 13 22:47 olcDatabase={-1}frontend.ldif
-rw------- 1 ldap ldap   695 Mar 13 22:47 olcDatabase={1}monitor.ldif
-rw------- 1 ldap ldap  1273 Mar 13 22:47 olcDatabase={2}bdb.ldif

#现使用openldap2.3配置文件的使用习惯
cp /usr/share/openldap-servers/slapd.conf.obsolete /etc/openldap/slapd.conf

#生产ldap管理员密码
slappasswd -s one | sed -e "s#{SSHA}#rootpw\t{SSHA}#g" >> /etc/openldap/slapd.conf #one是密码哦

#备份
cd /etc/openldap
cp slapd.conf slapd.conf.ori

#编辑
vi slapd.conf
#注释
    #指定使用的数据库
    114 #database       bdb
    #指定搜索后缀    
    115 #suffix         "dc=my-domain,dc=com"
    116 #checkpoint     1024 15
    #指定管理员DN
    117 #rootdn         "cn=Manager,dc=my-domain,dc=com"

#add
database        bdb                           
suffix          "dc=one,dc=com"
rootdn          "cn=admin,dc=one,dc=com"
#此时管理员为 admin 密码 one

#注释
     97 # enable on-the-fly configuration (cn=config)
     98 #database config
     99 #access to *
    100 #       by dn.exact="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage
    101 #       by * none
    102
    103 # enable server status monitoring (cn=monitor)
    104 #database monitor
    105 #access to *
    106 #       by dn.exact="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" read
    107 #        by dn.exact="cn=Manager,dc=my-domain,dc=com" read
    108 #        by * none
#add
access to *
    by self write
    by anonymous auth
    by * read


#add in the end
#日志级别
loglevel        296
#ldap缓存的记录数
cachesize       1000
#没达到2048k或者每10分钟做一次回写
checkpoint      2048 10




#slapd.conf
include     /etc/openldap/schema/corba.schema
include     /etc/openldap/schema/core.schema
include     /etc/openldap/schema/cosine.schema
include     /etc/openldap/schema/duaconf.schema
include     /etc/openldap/schema/dyngroup.schema
include     /etc/openldap/schema/inetorgperson.schema
include     /etc/openldap/schema/java.schema
include     /etc/openldap/schema/misc.schema
include     /etc/openldap/schema/nis.schema
include     /etc/openldap/schema/openldap.schema
include     /etc/openldap/schema/ppolicy.schema
include     /etc/openldap/schema/collective.schema
allow bind_v2
pidfile     /var/run/openldap/slapd.pid
argsfile    /var/run/openldap/slapd.args
TLSCACertificatePath /etc/openldap/certs
TLSCertificateFile "\"OpenLDAP Server\""
TLSCertificateKeyFile /etc/openldap/certs/password
access to *
    by self write
    by anonymous auth
    by * read
database        bdb
suffix          "dc=one,dc=com"
rootdn          "cn=admin,dc=one,dc=com"
directory   /var/lib/ldap
index objectClass                       eq,pres
index ou,cn,mail,surname,givenname      eq,pres,sub
index uidNumber,gidNumber,loginShell    eq,pres
index uid,memberUid                     eq,pres,sub
index nisMapName,nisMapEntry            eq,pres,sub
rootpw  {SSHA}9oY8eamAbmI712jtIeqkbaXg2ttM5Pfm
loglevel        296
cachesize       1000
checkpoint      2048 10

```

>权限说明 http://www.openldap.org/doc/admin24/access-control.html

## 配置rsyslog
```shell
cp /etc/rsyslog.conf /etc/rsyslog.conf.ori
echo "local4.*               /var/log/ldap.log" >> /etc/rsyslog.conf
/etc/init.d/rsyslog restart
```


## 配置ldap数据库
```shell
grep directory /etc/openldap/slapd.conf
# Do not enable referrals until AFTER you have a working directory
# The database directory MUST exist prior to running slapd AND
directory       /var/lib/ldap


cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown ldap:ldap /var/lib/ldap/DB_CONFIG
chmod 700 /var/lib/ldap/

#测试配置文件
slaptest -u
config file testing succeeded

```

## 启动ldap master
```shell
/etc/init.d/slapd start

netstat -naltup | grep 389
tcp        0      0 0.0.0.0:389                 0.0.0.0:*                   LISTEN      2300/slapd
tcp        0      0 :::389                      :::*                        LISTEN      2300/slapd
```
>启动服务官方说明: http://www.openldap.org/doc/admin24/runningslapd.html

opldap查询
===

```shell
ldapsearch -LLL -W -x -H ldap://one.com -D "cn=admin,dc=one,dc=com" -b "dc=one,dc=com" "(uid=*)"
Enter LDAP Password:
ldap_bind: Invalid credentials (49)
```
以上问题解决(2.3;2.4配置冲突问题)
```shell
rm -rf /etc/openldap/slapd.d/*    #删除默认的2.4配置文件
slaptest -f /etc/openldap/slapd.conf -F /etc/openldap/slapd.d  #生产新的配置文件
chown -R ldap:ldap /etc/openldap/slapd.d/
/etc/init.d/slapd restart

ldapsearch -LLL -W -x -H ldap://one.com -D "cn=admin,dc=one,dc=com" -b "dc=one,dc=com" "(uid=*)"
Enter LDAP Password:
No such object (32)
```
## 初始化ldap
```shell
#base.ldif
dn: dc=one,dc=com
objectClass: organization
objectClass: dcObject
dc: one
o: one

dn: ou=People,dc=one,dc=com
objectClass: organizationalUnit
ou: People

dn: ou=group,dc=one,dc=com
objectClass: organizationalUnit
ou: group

dn: cn=tech,ou=group,dc=one,dc=com
objectClass: posixGroup
description: 5oqA5pyv6YOo
gidNumber: 10001
cn: tech


#test_user.ldif
dn: uid=oldgirl,ou=People,dc=one,dc=com
objectClass: posixAccount
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
homeDirectory: /home/oldgirl
loginShell: /bin/bash
uid: oldgirl
cn: oldgirl
userPassword: 9oY8eamAbmI712jtIeqkbaXg2ttM5Pfm
uidNumber: 10005
gidNumber: 10001
sn: oldgirl


#导入数据
ldapadd -x -H ldap://one.com -D "cn=admin,dc=one,dc=com" -W -f base.ldif
ldapadd -x -H ldap://one.com -D "cn=admin,dc=one,dc=com" -W -f test_user.ldif

#查询
ldapsearch -LLL -w one -x -H ldap://one.com -D "cn=admin,dc=one,dc=com" -b "dc=one,dc=com" "(uid=*)"

dn: uid=oldgirl,ou=Peopel,dc=one,dc=com
objectClass: posixAccount
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
homeDirectory: /home/oldgirl
loginShell: /bin/bash
uid: oldgirl
cn: oldgirl
userPassword:: OW9ZOGVhbUFibUk3MTJqdEllcWtiYVhnMnR0TTVQZm0=
uidNumber: 10005
gidNumber: 10001
sn: oldgirl


#部分参数
-LLL       print responses in LDIF format without comments
-x         Simple authentication
```

## ldap客户端管理
>web管理工具下载地址:https://www.ldap-account-manager.org/lamcms/

```shell
yum install -y httpd php php-ldap php-gd -y

tar zxf ldap-account-manager-3.7.tar.gz
cd ldap-account-manager-3.7/config
cp config.cfg_sample config.cfg
cp lam.conf_sample lam.conf

sed -i 's#cn=Manager#cn=admin#g' lam.conf
sed -i 's#dc=my-domain#dc=one#g' lam.conf
diff lam.conf_sample lam.conf

mv ldap-account-manager-3.7 /var/www/html/ldap
chown -R apache:apache /var/www/html/ldap/
```

SASL验证机制
===

>Simple Authentication and Security Layer

```shell
yum install -y *sasl*
```
## 查看验证机制
```shell
grep -i mech /etc/sysconfig/saslauthd 
# Mechanism to use when checking passwords.  Run "saslauthd -v" to get a list
# of which mechanism your installation was compiled with the ablity to use.
MECH=pam
# Options sent to the saslauthd. If the MECH is other than "pam" uncomment the next line.

#替换验证机制为shadow(本地认证)
sed -i 's#MECH=pam#MECH=shadow#g' /etc/sysconfig/saslauthd

#重启saslauthd
/etc/init.d/saslauthd restart

#测试认证
testsaslauthd -uroot -proot123
0: OK "Success."
```

## 配置ldap认证
```
sed -i 's#MECH=shadow#MECH=ldap#g' /etc/sysconfig/saslauthd
/etc/init.d/saslauthd restart

vi /etc/saslauthd.conf
#add
ldap_servers: ldap://one.com/
#ldap_uri: ldap://ldap.rone.one.com
#ldap_version: 3
#ldap_start_tls: 0
ldap_bind_dn: cn=admin,dc=one,dc=com
ldap_bind_pw: one
ldap_search_base: ou=People,dc=one,dc=com
ldap_filter: uid=%U
#ldap_filter: mail=%U@one.com
ldap_password_attr: userPassword
#ldap_sasl: 0

#验证
testsaslauthd -uone -proot123
```

SVN通过ldap认证
===

```shell
vi /etc/sasl2/svn.conf
#add
pwcheck_method: saslauthd
mech_list: PLAIN LOGIN

/etc/init.d/saslauthd restart

#修改SVN的配置使其支持ldap认证
vi svnserve.conf
#change
# use-sasl = true
#to
use-sasl = true

#or use sed
sed -i 's@# use-sasl = true@use-sasl = true@g' svnserve.conf
```
