---
author: asmbits
comments: true
date: 2018-04-21 14:42:00 +0000
layout: post
title: mysql 备份与恢复(使用XtraBackup)
image: /assets/images/blog_back.jpg
categories:
- database
tags:
- mysql
---

安装(centos6)
===

```shell
wget https://www.percona.com/downloads/XtraBackup/Percona-XtraBackup-2.3.2/binary/redhat/6/x86_64/percona-xtrabackup-2.3.2-1.el6.x86_64.rpm

wget http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm

rpm -ivh epel-release-6-8.noarch.rpm

yum -y install perl perl-devel libaio libaio-devel perl-Time-HiRes perl-DBD-MySQL 

yum --enablerepo=epel localinstall percona-xtrabackup-2.3.2-1.el6.x86_64.rpm -y
```

<!-- more -->

全备及恢复
===

- 恢复数据库之后,要注意目录和文件的权限
- 备份结束后,会自动显示binlog文件及位置,以便做从库使用

## 全备
```shell
innobackupex --defaults-file=/opt/mysql/my.cnf  --user=root --password=*** /backup/mysql/data 

#--defaults-file mysql配置文件
#--user 数据库有管理员权限的用户
#--password 数据库管理员密码
#最后是备份目录
```

## 恢复

**恢复之前，要先关闭数据库，并且删除数据文件和日志文件**

```shell
innobackupex --defaults-file=/opt/mysql/my.cnf \
--user=root --password=*** --use-memory=4G \
--apply-log /backup/mysql/data/2013-10-29_09-05-25


innobackupex --defaults-file=/opt/mysql/my.cnf \
--user=root --password=***  \
--copy-back /backup/mysql/data/2013-10-29_09-05-25

#--defaults-file mysql配置文件
#--user 数据库有管理员权限的用户
#--password 数据库管理员密码
#最后是备份目录
```

- 恢复分两步
  - 1. --apply-log 保持事物一致性
  - 2. --copy-back 恢复整个数据库


增量备份及恢复
===

*增量备份仅针对InnoDB这类支持事务的引擎,对于MyISAM等引擎,则仍然是全备*

```shell
#增量备份需要基于全备,
#先假设我们已经有了一个全备(/backup/mysql/data/2013-10-29_09-05-25)
#我们需要在该全备的基础上做增量备份
innobackupex --defaults-file=/opt/mysql/my.cnf  --user=root --password=*** --incremental-basedir=/backup/mysql/data/2013-10-29_09-05-25 --incremental /backup/mysql/data

#--defaults-file mysql配置文件
#--user 数据库有管理员权限的用户
#--password 数据库管理员密码
#--incremental-basedir 已经存在的全备份目录
#--incremental 增量备份目录
```

上面语句执行成功之后,会在--incremental执行的目录下创建一个时间戳子目录(本例中为：/backup/mysql/data/2013-10-29_09-52-37),在该目录下存放着增量备份的所有文件.

## 增量备份说明
在备份目录下,有一个文件xtrabackup_checkpoints记录着备份信息,全备的信息如下:
```
backup_type = full-backuped  
from_lsn = 0  
to_lsn = 563759005914  
last_lsn = 563759005914  
```
基于该全备的增量备份的信息如下:
```
backup_type = incremental  
from_lsn = 563759005914  
to_lsn = 574765133284  
last_lsn = 574765133284  
```

**从上面可以看出,增量备份的from_lsn正好等于全备的to_lsn.**

### 可以使用基于增量备份的增量备份
把--incremental-basedir设置为上一次增量备份的目录即可,如下所示：
```shell
innobackupex --defaults-file=/opt/mysql/my.cnf  --user=root --password=*** --incremental-basedir=/backup/mysql/data/2013-10-29_09-52-37 --incremental /backup/mysql/data  
```
它的xtrabackup_checkpoints记录着备份信息如下：
```
backup_type = incremental  
from_lsn = 574765133284  
to_lsn = 574770200380  
last_lsn = 574770200950  
```
可以看到,该增量备份的from_lsn是从上一次增量备份的to_lsn开始的

## 增量恢复

**恢复之前，要先关闭数据库，并且删除数据文件和日志文件**

### 第一步,合并全备和所以增量
```
innobackupex --defaults-file=/etc/my.cnf --user=root --password= --use-memory=4G --apply-log --redo-only BASE-DIR
    
innobackupex --defaults-file=/etc/my.cnf --user=root --password= --apply-log --redo-only BASE-DIR --incremental-dir=INCREMENTAL-DIR-1  

innobackupex --defaults-file=/etc/my.cnf --user=root --password= --apply-log BASE-DIR --incremental-dir=INCREMENTAL-DIR-2  


#--use-memory=4G 可以提高速度
#BASE-DIR 全备目录
#INCREMENTAL-DIR-1 第一次的增量备份
#INCREMENTAL-DIR-2 第二次的增量备份,以此类推,基于第一次增量备份.
```

**最后一步的增量备份并没有--redo-only选项!**

* 可以使用--use_memory提高性能
* 以上语句执行成功之后,最终数据在BASE-DIR(即全备目录)下.

### 第二步,恢复数据库
#### 保持事物一致性
```shell
innobackupex --defaults-file=/etc/my.cnf --user=root --password= --apply-log BASE-DIR  
```

#### 恢复数据库
*BASE-DIR里的备份文件已完全准备就绪*
```shell
innobackupex --defaults-file=/etc/my.cnf --user=root --password= --copy-back BASE-DIR  
```




