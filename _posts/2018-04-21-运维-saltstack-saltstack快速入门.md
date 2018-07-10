---
author: ucrux
comments: true
date: 2018-04-21 21:35:00 +0000
layout: post
title: saltstack快速入门
image: /assets/images/blog_back.jpg
categories:
- 运维
tags:
- saltstack
---

**本文档基于centos6**

```
                发送任务
            | 4505 -------> |
salt-master |               | salt-minion
            | 4506 <------- |
                返回结果

+-------------------+---------------+
| IP                | memo          |
+-------------------+---------------+
| 192.168.122.10    | salt-master   |
| 192.168.122.101   | salt-minion01 |
| 192.168.122.102   | salt-minion02 |
+-------------------+---------------+
```

salt安装
===

```shell
#安装epel源
rpm -Uvh  http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
```

## master install
```shell
yum install -y salt-master
```

## minion install
```shell
yum install -y salt-minion
```

## 修改minion端配置文件
```shell
vi +16 /etc/salt/minion
#master: salt
#so we add "192.168.122.10  salt" in /etc/hosts
echo "192.168.122.10    salt" >> /etc/hosts
```

## 启动服务
### master
```shell
/etc/init.d/salt-master start

#vi +679 /etc/salt/master
#log_level: warning         #可以修改日志级别 #master上修改配置文件不需要重启,立即生效
```

### minion
```shell
/etc/init.d/salt-minion start
```

salt master 相关操作
===
```shell
salt-key -h         #自行查看帮助

#列出已经发现的客户端的key
salt-key -L
Accepted Keys:
Denied Keys:
Unaccepted Keys:
heartbeat01
heartbeat02
Rejected Keys:

#添加相应的客户端key
salt-key -a heartbeat01
#or salt-key -A -y #添加所有

#测试
salt '*' test.ping
heartbeat02:
    True
heartbeat01:
    True

```

*此时的客户端*
```
/etc/salt/
├── minion
├── minion.d
│   └── _schedule.conf
├── minion_id     #这个文件记录的 minion_id: heartbeat01 or heartbeat02
└── pki
    └── minion
       ├── minion_master.pub     #master 发送过来的公钥,
       ├── minion.pem            #minion 的私钥
       └── minion.pub            #minion 的公钥
```

**修改已加入salt-mastre的minion的id**
```shell
#在master中操作
salt-key -d heartbeat01 -y

#在minion中操作
hostname minion01
sed -i 's@HOSTNAME=heartbeat01@HOSTNAME=minion01@g' /etc/sysconfig/network
rm -rf /etc/salt/minion_id /etc/salt/pki
/etc/init.d/salt-minion restart

#在master中操作
salt-key -A -y
```

**key在mater上的保存位置**

```
/etc/salt/pki/master/
├── master.pem
├── master.pub
├── minions
│   ├── heartbeat01
│   ├── heartbeat02
│   └── resin01
├── minions_autosign
├── minions_denied
├── minions_pre
└── minions_rejected
```

**使用salt master 重启minion的服务**

```shell
salt "*" service.restart salt-minion
```

returner
===

**salt执行结果**

## 将返回结果放到系统日志中
```shell
salt "*" test.ping --return syslog      #结果保存到minior的/var/log/messages中

#Apr 22 16:15:44 heartbeat01 python2.6: {"fun_args": [], "jid": "20170422161545074967", "return": true, "retcode": 0, "success": true, "fun": "test.ping", "id": "heartbeat01"}
```

## 将返回结果放到mysql中

```
salt-master ---------> salt-minion -----------> mysql server
```

### minion端
```
#add in /etc/salt/minion
mysql.host: '192.168.122.10'
mysql.user: 'salt'
mysql.pass: 'salt123'
mysql.db: 'salt'
mysql.port: 3306

#安装MySQL-python模块
```

### 数据库
```sql
#创建数据库
create database `salt`
    default character set utf8
    default collate utf8_general_ci ;

#创建用户
grant all privileges on salt.* to salt@'%' identified by 'salt123' ;

#创建表
use salt;

drop table if exists `jids`;
create table `jids` (
    `jid` varchar(255) not null,
    `load` mediumtext not null,
    unique key `jid` (`jid`)
) engine=InnoDB default charset=utf8 ;

drop table if exists `salt_returns` ;
create table `salt_returns` (
    `fun` varchar(50) not null ,
    `jid` varchar(255) not null ,
    `return` mediumtext not null ,
    `id` varchar(255) not null,
    `success` varchar(10) not null,
    `full_ret` mediumtext not null,
    `alter_time` timestamp default current_timestamp,
    key `id` (`id`) ,
    key `jid` (`jid`),
    key `fun` (`fun`)
) engine=InnoDB default charset=utf8;

```

### 测试
```shell
salt "*" cmd.run 'hostname' --return mysql
```

event
===

zeroMQ PUB interface

```
    salt-master -----------> salt-minion
        |
        |
        |
        V
    mysql server
```

## 通过event获取执行结果
salt-master
```python
import  salt.utils.event
event = salt.utils.event.MasterEvent('/var/run/salt/master')

for eachevent in event.iter_events(full=True) :
    print eachevent
    print "------"
```

## master 安装必须软件
```shell
yum install mysql-server mysql -y
yum install MySQL-python

/etc/init.d/mysql start
```

**创建数据库,表及用户授权如上**

## event写入数据库

```python
#!/bin/evn python
#coding=utf8

# import python libs
import json

# import salt modules
import salt.config
import salt.utils.event

# import third part libs
import MySQLdb

__opts__ = salt.config.client_config('/etc/salt/master')

# create mysql connect
conn = MySQLdb.connect( host=__opts__['mysql.host'], user=__opts__['mysql.user'], \
                        passwd=__opts__['mysql.pass'],db=__opts__['mysql.db'], port=__opts__['mysql.port'] )
cursor = conn.cursor()

# listen salt master event system
event = salt.utils.event.MasterEvent(__opts__['sock_dir'])

for eachevent in event.iter_events(full=True) :
    ret = eachevent['data']
    if "salt/job" in eachevent['tag'] :
        # return event
        if ret.has_key('id') and ret.has_key('return') :
            # igonre saltutil.find_job event
            if ret['fun'] == 'saltutil.find_job' :
                continue

            sql = "insert into `salt_returns` (`fun`, `jid`, `return`, `id`, `success`, `full_ret` ) \
                   values ( %s, %s, %s, %s, %s, %s )"

            cursor.execute( sql, ( ret['fun'], ret['jid'], json.dumps(ret['return']), ret['id'], ret['success'], json.dumps(ret) ) )
            cursor.execute("commit")
    else :
        pass
```

```
#add in /etc/salt/master
mysql.host: '192.168.122.109'
mysql.user: 'salt'
mysql.pass: 'salt123'
mysql.db: 'salt'
mysql.port: 3306
```

分组
===

| letter | 含义           | 例子                                               |        |            |
| ------ | -------------- | -------------------------------------------------- | ------ | ---------- |
| G      | Grains glob    | G@os:Ubuntu                                        |        |            |
| E      | PCRE Minion id | E@web\d+\.(dev                                     | qa     | prod)\.loc |
| P      | Grains PCRE    | P@os:(RedHat                                       | Fedora | CentOS)    |
| L      | minion list    | L@minion01.one.com,minion02.one.com or bl*.one.com |        |            |
| I      | Pillar glob    | I@pdata:foobar                                     |        |            |
| S      | subnet/ip      | S@192.168.122.0/24 or S@192.168.122.10             |        |            |
| R      | range cluster  | R@%foo.bar                                         |        |            |
| D      | minion data    | D@key:value                                        |        |            |


```shell
vi +710 /etc/salt/master

#nodegroups:
#  group1: 'L@foo.domain.com,bar.domain.com,baz.domain.com and bl*.domain.com'
#  group2: 'G@os:Debian and foo.domain.com'
```

模块
===

**查看帮助**
salt "heartbeat01" sys.doc

文件系统
===

```shell
vi +416 /etc/salt/master

file_roots:
  base:
      - /srv/salt


mkdir -p /srv/salt
mkdir -pv /srv/salt/script

vi /srv/salt/script/fuck.sh

#!/bin/sh
while true
do
    sleap 1
    echo 1 >> /tmp/log
done


#运行
salt '*' cmd.script salt://script/fuck.sh
```

## sls文件
```shell
vi /srv/salt/top.sls

base:               #file_roots name
 '*':               #将被操作的机器
  - hosts           #使用的sls文:/srv/salt/hosts.sls

vi /srv/salt/hosts.sls
/tmp/hosts:
 file.managed:
  - source: salt://etc/hosts
  - user: root
  - group: root
  - mode: 600
```
*将master /srv/salt/etc/hosts 同步到minion 的 /tmp/hosts*

### 执行sls
```shell
salt "*" state.highstate

#其他方式
mkdir -pv /srv/salt/hosts
mv /srv/salt/hosts.sls /srv/salt/hosts

salt "*" state.sls hosts.hosts

##or

mv /srv/salt/hosts/hosts.sls /srv/salt/hosts/init.sls
salt "*" state.sls hosts

```

Grains
===

**可以使用grains进行主机匹配**

获取主机信息
```shell
salt "*" grains.ls
salt "*" grains.items
salt "*" grains.item os
```
## 自定义grains
在 minion 端修改
vi /etc/salt/minion
```
grains:
 roles:
  - webserver
  - memcache
 deployment: datacenter4
 cabinet: 13
 cab_u: 14-15
```
/etc/init.d/salt-minion restart     #重启minion

> yaml文档 http://docs.saltstack.cn/topics/yaml/index.html
> state模块文档 http://docs.saltstack.com/en/latest/ref/states/all/index.html


使用salt安装nginx
===
sls文件
```shell
mkdir /srv/salt/nginx
vi /srv/salt/nginx/init.sls

nginx:
  pkg:
    - installed
  service:
   - running
   - enable: True
   - realod: True
   - watch:                                     #监控以下两个文件,如果有变动,就使用service模块重启
     - pkg: nginx
     - file: /etc/nginx/nginx.conf
     - file: /etc/nginx/conf.d/default.conf

/etc/nginx/nginx.conf:
  file.managed:
    - source: salt://etc/nginx/nginx.conf
    - user: root
    - group: root
    - mode: 644

/etc/nginx/conf.d/default.conf:
  file.managed:
    - source: salt://etc/nginx/conf.d/default.conf
    - user: root
    - group: root
    - mode: 644
```

执行安装
```shell
salt "*" state.sls nginx
```

Pillar
===
定时执行
```
schedule:
  highstate:
    function: state.highstate
    minutes:1
```
> 文档 http://docs.saltstack.cn/topics/jobs/schedule.html

```shell
vi +529 /etc/salt/master

pillar_roots:
  base:
      - /srv/pillar

######################
mkdir /srv/pillar

cd /srv/pillar
vi top.sls

base:                       #指向 /srv/pillar
  "*":
    - nginx

######################
mkdir nginx
cd nginx
vi init.sls

schedule:
  nginx:
    function: state.sls
    minutes: 1
    args:
      - 'nginx'
```

```shell
#查看pillar
salt "*" pillar.data
```
```shell
#执行pillar
salt "*" saltutil.refresh_pillar
```

proxy 代理
===
```
                                master   ------------------------------> mysql
                               --------
                             /         \
                  -----------           ----------------
                  |                                    |
                  |                                    |
                proxy                                proxy
               -------                              -------
               |     |                              |     |
               |     |                              |     |
               |     |                              |     |
              /       \                            /       \
          minon1    minion2                   minion3    minion4



salt syndic   <--->    master
   |
   |
   V
 minion
```

salt syndic(proxy) 也是一个 master,只不过要连接更高层的master

## 在proxy上操作
### 安装
```shell
yum install -y salt-master salt-syndic  #有些版本的salt-syndic包含在salt-master中
```

### 配置文件
```shell
vi /etc/salt/master

syndic_master: salt                 #更高级别的master
syndic_log_file:    /var/salt/log   #日志文件
order_master: True                  #不加这个值,则不进行转发
```
### 启动服务
```shell
/etc/init.d/salt-syndic start
/etc/init.d/salt-master start

salt-key -A -y
```

## 在master上操作
```shell
#添加代理的key
salt-key -A -y

#删除原来minion的key,因为要让proxy来管理
salt-key -d heartbeat01 -y
salt-key -d heartbeat02 -y
```

**然后minion在配置文件中指向proxy即可**
