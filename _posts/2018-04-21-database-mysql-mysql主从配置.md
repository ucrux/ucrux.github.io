---
author: asmbits
comments: true
date: 2018-04-21 14:40:00 +0000
layout: post
title: mysql主从配置
image: /assets/images/blog_back.jpg
categories:
- database
tags:
- mysql
---

```
    master                       slave
--------------               -------------
|            |               |           |    change master...
|            |               |           |      master_host
|   IO       |               |    SQL    |      master_user
| bin-log    |               |   |       |      master_password
|            |               |   | IO    |      master_port
|            |               | ralay-log |      master_file
|            |               |           |      master_log_file
|            |               |           |      master_log_pos
  /                                            
 /                                            start slave 
/ request master                    master.info
guest
```

<!-- more -->

## 配置

### 主节点必须打开binlog

```shell
egrep 'log-bin|server-id' 3306/my.cnf 
  log-bin             = /data/3306/mysql-bin
  server-id           = 1

mysql -uroot -prage1984 -S 3306/mysql.sock -e "show variables like 'log_bin';"
  +---------------+-------+
  | Variable_name | Value |
  +---------------+-------+
  | log_bin       | ON    |
  +---------------+-------+
```
### 创建同步账户
```sql
grant replication slave on *.* to 'rep'@'10.0.1.9' identified by 'aDgc9tSDK8dQkSWs';

flush privileges;
```

### 备份主节点
```shell
mysqldump -uroot -prage1984 -S /data/3307/mysql.sock -A --master-data=1 > /tmp/3307bak.sql
```

### 在从节点恢复主节点备份, 启动从节点
```shell
mysql -uroot -p < 3307bak.sql
```

- 由于备份的时候使用了--master-data=1参数,所以在备份文件中会有主节点备份时binlog的位置
- 以下操作在从节点操作

```sql
CHANGE MASTER TO
MASTER_HOST='192.168.122.100',
MASTER_PORT=3306,
MASTER_USER='rep',
MASTER_PASSWORD='root123',
MASTER_LOG_FILE='mysql-bin.000018',
MASTER_LOG_POS=1330;
```


## 查看主从状态
### master
```sql
show processlist\G;

*************************** 1. row ***************************
     Id: 1
   User: rep
   Host: 192.168.122.100:49499
     db: NULL
Command: Binlog Dump
   Time: 1491
  State: Master has sent all binlog to slave; waiting for binlog to be updated
   Info: NULL
```


### slave
```sql
show processlist\G ;
*************************** 1. row ***************************
     Id: 1
   User: system user
   Host: 
     db: NULL
Command: Connect
   Time: 994
  State: Slave has read all relay log; waiting for the slave I/O thread to update it
   Info: NULL
```

### 注意事项
如果主库开启了 gtid_mode = on 那么从库的 my.cnf 就要添加如下参数
```
gtid_mode                   = on
enforce_gtid_consistency    = on
#skip_slave_start           = 1         #这个参数在第一次start slave后就不需要了
server-id                   = 999

log-bin                     = /data/mysql/logs/binlog/mysql-bin
binlog_format               = mixed
log-slave-updates           = 1         #从库需要开启 relay-log
relay-log                   = /data/mysql/logs/binlog/relay-bin
relay-log-info-file         = /data/mysql/logs/binlog/relay-log.info
```

## 其他说明

### 从库只读
在从库my.cnf中添加
```
[mysqld]
read-only
```

### 排除不需要同步的库
在主库my.cnf中添加
```sql
[mysqld]
binlog-ignore-db=mysql
binlog-ignore-db=performance_schema
binlog-ignore-db=information_schema
```

### 同步错误处理
#### 跳过一条sql
在从库上操作
```sql
stop slave
set global sql_slave_skip_counter=1;
start slave
```

#### 忽略指定sql错误
在从库上操作
```sql
[mysqld]
slave-skip-errors=1032,1062,1007
```

#### 从库打开binlog
```sql
[mysqld]
log-bin                         = /data/3307/mysql-bin
log-slave-updates
expire_logs_days                = 7   #7days binlog will be saved
```

#### 停止从库
```sql
stop slave;
reset master;
quit;
```

## 双主
### master1修改配置文件(my.cnf)
```sql
[mysqld]
auto_increment_increment = 2             #解决自增主键冲突问题
auto_increment_offset    = 1
log-bin = /data/3306/mysql-bin
log-slave-updates
```
### master2修改配置文件(my.cnf)
```sql
[mysqld]
auto_increment_increment = 2             #解决自增主键冲突问题
auto_increment_offset    = 2
log-bin = /data/3307/mysql-bin
log-slave-updates
```

### 备份 master2
```shell
mysqldump -uroot -prage1984 -S /data/3307/mysql.sock -A -B --master-data=1 -x --events > /tmp/3307bak.sql
```

### master1执行change master
```sql
change master to
master_host='192.168.122.100',
master_port=3307,
master_user='rep',
master_password='root123';
```


