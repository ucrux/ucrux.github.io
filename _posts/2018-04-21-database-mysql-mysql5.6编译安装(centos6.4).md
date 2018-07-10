---
author: ucrux
comments: true
date: 2018-04-21 14:38:00 +0000
layout: post
title: mysql5.6编译安装(centos6.4)
image: /assets/images/blog_back.jpg
categories:
- database
tags:
- mysql
---

## 卸载旧版本

使用下面的命令检查是否安装有MySQL Server
```shell
rpm -qa | grep mysql
```
有的话通过下面的命令来卸载掉
```shell
#普通删除模式
rpm -e mysql
#强力删除模式,如果使用上面命令删除时,提示有依赖的其它文件,则用该命令可以对其进行强力删除
rpm -e --nodeps mysql
```

<!-- more -->

## 编译安装MySQL
### 安装编译代码需要的包
```shell
yum -y install make gcc-c++ cmake bison-devel  ncurses-devel
```
### 下载MySQL 5.6.37
```shell
wget http://ftp.ntu.edu.tw/MySQL/Downloads/MySQL-5.6/mysql-5.6.37.tar.gz
tar xf mysql-5.6.37.tar.gz
cd mysql-5.6.37
```
### 编译安装
```shell
cmake \
-DCMAKE_INSTALL_PREFIX=/opt/mysql-5.6.37 \
-DMYSQL_DATADIR=/data/mysql/data \
-DMYSQL_UNIX_ADDR=/data/mysql/sock/mysql.sock \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DEXTRA_CHARSETS=gbk,gb2312,utf8,ascii \
-DENABLED_LOCAL_INFILE=ON \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_PARTITION_STORAGE_ENGINE=1 \
-DWITH_FEDERATED_STORAGE_ENGINE=1 \
-DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
-DWITHOUT_EXAMPLE_STORAGE_ENGINE=1 \
-DWITH_FAST_MUTEXES=1 \
-DWITH_ZLIB=1 \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_READLINE=1 \
-DWITH_EMBEDDED_SERVER=1 \
-DWITH_DEBUG=0 

make -j4 && make install

# 将mysql的库加入到ld路径中
vi /etc/ld.so.conf.d/mysql.conf
#add
/opt/mysql/lib

ldconfig
```

><p>编译的参数可以参考 <a href="http://dev.mysql.com/doc/refman/5.5/en/source-configuration-options.html">编译选项</p>


## 配置MySQL

### 设置权限
使用下面的命令查看是否有mysql用户及用户组
```shell
cat /etc/passwd  #查看用户列表
cat /etc/group   #查看用户组列表
```
如果没有就创建
```shell
groupadd -g 500 mysql                               #gid=500
useradd -g mysql -u 500 mysql -s /sbin/nologin      #uid=500
```
修改/usr/local/mysql权限
```shell
chown -R mysql:mysql /opt/mysql
```
### 初始化配置

**最好是在修改my.cnf之后再初始化数据库**

进入安装路径
```shell
cd /opt/mysql
```
进入安装路径,执行初始化配置脚本,创建系统自带的数据库和表
```shell
scripts/mysql_install_db --basedir=/opt/mysql --datadir=/mysqldata --user=mysql
```

- 注:在启动MySQL服务时,会按照一定次序搜索my.cnf,先在/etc目录下找,找不到则会搜索"$basedir/my.cnf",在本例中就是 /usr/local/mysql/my.cnf,这是新版MySQL的配置文件的默认位置！
- 在CentOS 6.4版操作系统的最小安装完成后,在/etc目录下会存在一个my.cnf,需要将此文件更名为其他的名字,如：/etc/my.cnf.bak,否则,该文件会干扰源码安装的MySQL的正确配置,造成无法启动.
- 在使用"yum update"更新系统后,需要检查下/etc目录下是否会多出一个my.cnf,如果多出,将它重命名成别的.否则,MySQL将使用这个配置文件启动,可能造成无法正常启动等问题.

添加服务,拷贝服务脚本到init.d目录,并设置开机启动
```shell
cp support-files/mysql.server /etc/init.d/mysqld
chkconfig mysqld on
service mysqld start  #启动MySQL
```
添加PATH路径
```shell
vi /etc/profile
# add
PATH=/opt/mysql/bin:$PATH
export PATH

source /etc/profile
```
MySQL启动成功后,root默认没有密码,我们需要设置root密码.<br>
执行下面的命令修改root密码
```shell
mysql -uroot  
mysql> SET PASSWORD = PASSWORD('mysql');
```

若要设置root用户可以远程访问,执行
```shell
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'172.16.%' IDENTIFIED BY 'password' WITH GRANT OPTION;
```
>其中password为远程访问时,root用户的密码,可以和本地不同.

配置防火墙
```shell
vi /etc/sysconfig/iptables
# add
-A INPUT -m state --state NEW -m tcp -p -dport 3306 -j ACCEPT

service iptables restart
```


## 多实例启动

- pid-file
- socket
- log-error

以上文件及其他可能引起冲突的文件位置一定不能一样

### 多实例启动\停止
```shell
mysqld_safe --defaults-file=/data/3307/my.cnf
mysqladmin -uroot -prage1984 -S /data/3306/mysql.sock shutdown
```

### 多实例登录
```shell
mysql -S /data/3306/mysql.sock
mysql> system mysql -S /data/3307/mysql.sock
```

### 多实例改密码
```shell
mysqladmin -S /data/3306/mysql.sock password 'rage1984'
```
