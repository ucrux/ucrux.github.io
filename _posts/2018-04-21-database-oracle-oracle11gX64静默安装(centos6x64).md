---
author: asmbits
comments: true
date: 2018-04-21 14:46:00 +0000
layout: post
title: oracle11gx64 在centos6x64上静默安装
image: /assets/images/blog_back.jpg
categories:
- database
tags:
- oracle
---

## 检查安装环境

- mininal : 1GB of RAM
- recommened : 2GB or more

<!-- more -->

```shell
grep MemTotal /proc/meminfo
```

## 安装所需要的包

- 此处版本号可能不同
  - binutils-2.20.51.0.2-5.11.el6 (x86_64)
  - compat-libcap1-1.10-1 (x86_64)
  - compat-libstdc++-33-3.2.3-69.el6 (x86_64)
  - compat-libstdc++-33-3.2.3-69.el6.i686
  - gcc-4.4.4-13.el6 (x86_64)
  - gcc-c++-4.4.4-13.el6 (x86_64)
  - glibc-2.12-1.7.el6 (i686)
  - glibc-2.12-1.7.el6 (x86_64)
  - glibc-devel-2.12-1.7.el6 (x86_64)
  - glibc-devel-2.12-1.7.el6.i686
  - ksh
  - libgcc-4.4.4-13.el6 (i686)
  - libgcc-4.4.4-13.el6 (x86_64)
  - libstdc++-4.4.4-13.el6 (x86_64)
  - libstdc++-4.4.4-13.el6.i686
  - libstdc++-devel-4.4.4-13.el6 (x86_64)
  - libstdc++-devel-4.4.4-13.el6.i686
  - libaio-0.3.107-10.el6 (x86_64)
  - libaio-0.3.107-10.el6.i686
  - libaio-devel-0.3.107-10.el6 (x86_64)
  - libaio-devel-0.3.107-10.el6.i686
  - make-3.81-19.el6
  - sysstat-9.0.4-11.el6 (x86_64)

```shell
yum install gcc make binutils setarch compat-db libstdc++-devel unixODBA unixODBC-devel libaio-devel sysstat
```
需要单独安装的包,因为冲突的关系,所以要加参数–nodeps
```shell
# pdksh-5.2.14-36.el5.i386.rpm
rpm -ivh --nodeps pdksh
```

## 配置sysctl.conf
```shell
vim /etc/sysctl.conf  
  # Controls the maximum shared segment size, in bytes
  #kernel.shmmax = 68719476736
  kernel.shmmax = 4294967295
    
  # Controls the maximum number of shared memory segments, in pages
  #kernel.shmall = 4294967296
  kernel.shmall = 268435456
  fs.file-max = 65535
    
  fs.inotify.max_user_watches=892000
    
  #Below for oracle11g
  fs.aio-max-nr = 1048576
  kernel.core_uses_pid = 1
  kernel.shmmax = 536870912
  kernel.shmmni = 4096
  kernel.shmall = 2097152
  kernel.sem = 250 32000 100 128
  net.core.rmem_default = 4194304
  net.core.rmem_max = 4194304
  net.core.wmem_default = 262144
  net.core.wmem_max = 262144
  fs.file-max = 6815744                                                                       
  net.ipv4.ip_local_port_range = 9000 65000
```

## 增加用户组及用户
```shell
groupadd dba
groupadd oinstall
useradd oracle -g oinstall -G dba
passwd oracle
    
mkdir -p /opt/oracle
chown -R oracle:dba /opt/oracle
chmod -R 755 /opt/oracle
```

### 修改oracle用户配置文件(.bash_profile)
    ORACLE_BASE=/opt/oracle
    ORACLE_HOME=$ORACLE_BASE/product/11.2.0/dbhome_1
    
    ORACLE_SID=one
    PATH=$ORACLE_HOME/bin:$PATH
    ORACLE_OWNER=oracle
    
    export ORACLE_UNQNAME=$ORACLE_SID
    export ORACLE_BASE ORACLE_HOME ORACLE_SID PATH ORACLE_OWNR
### 增加shell限制
为了提升性能增加oracle用户的shell限制.
```shell
vi /etc/security/limits.conf  
# (在文件最后增加或修改以下参数)
  oracle    soft    nproc   2047
  oracle    hard    nproc   16384
  oracle    soft    nofile  1024
  oracle    hard    nofile  65536

vi /etc/pam.d/login
# (在文件最后增加或修改以下参数)
  session    required     pam_limits.sosession
  session    required     /lib64/security/pam_limits.so

vi /etc/profile
# (在文件最后增加或修改以下脚本)
  if [ $USER = "oracle" ]; then
    if [ $SHELL = "/bin/sh" ]; then
      ulimit -p 16384
      ulimit -n 65536
    else
      ulimit -u 16384 -n 65536
    fi
  fi
```

## 修改host
```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 oradb
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6 oradb
```
## 修改安装响应文件
```shell
su - oracle
cp db_install.rsp db_install_swonly.rsp
vi db_install_swonly.rsp
  oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v11_2_0
  oracle.install.option=INSTALL_DB_SWONLY
  ORACLE_HOSTNAME=oradb
  UNIX_GROUP_NAME=oinstall
  INVENTORY_LOCATION=/opt/oracle/oraInventory
  SELECTED_LANGUAGES=en,zh_CN,zh_TW
  ORACLE_HOME=/opt/oracle/product/11.2.0/dbhome_1
  ORACLE_BASE=/opt/oracle
  oracle.install.db.InstallEdition=EE
  oracle.install.db.isCustomInstall=true
  oracle.install.db.customComponents=oracle.server:11.2.0.1.0,oracle.sysman.ccr:10.2.7.0.0,oracle.xdk:11.2.0.1.0,oracle.rdbms.oci:11.2.0.1.0,oracle.network:11.2.0.1.0,oracle.network.listener:11.2.0.1.0,oracle.rdbms:11.2.0.1.0,oracle.options:11.2.0.1.0,oracle.rdbms.partitioning:11.2.0.1.0,oracle.oraolap:11.2.0.1.0,oracle.rdbms.dm:11.2.0.1.0,oracle.rdbms.dv:11.2.0.1.0,orcle.rdbms.lbac:11.2.0.1.0,oracle.rdbms.rat:11.2.0.1.0
  oracle.install.db.DBA_GROUP=dba
  oracle.install.db.OPER_GROUP=oinstall
  DECLINE_SECURITY_UPDATES=true
```
## 静默安装
```shell
# 用oracle用户执行
./runInstaller -silent -responseFile /opt/oracle/database/response/db_install_one.rsp
```

## 修改好dbca.rsp,静默建库
```shell
dbca -silent -responseFile /oracle/software/database/response/dbca_one.rsp 
```

## 附录
### db_install_swonly.rsp
```
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v11_2_0
oracle.install.option=INSTALL_DB_SWONLY
ORACLE_HOSTNAME=DataBase
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/oracle/oraInventory
SELECTED_LANGUAGES=en,zh_CN,zh_TW
ORACLE_HOME=/oracle/product/11g
ORACLE_BASE=/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.EEOptionsSelection=true
oracle.install.db.optionalComponents=oracle.rdbms.partitioning:11.2.0.2.0,oracle.oraolap:11.2.0.2.0,oracle.rdbms.dm:11.2.0.2.0,oracle.rdbms.dv:11.2.0.2.0,orcle.rdbms.lbac:11.2.0.2.0
oracle.install.db.DBA_GROUP=dba
oracle.install.db.OPER_GROUP=oinstall
oracle.install.db.CLUSTER_NODES=
oracle.install.db.isRACOneInstall=
oracle.install.db.racOneServiceName=
oracle.install.db.config.starterdb.type=
oracle.install.db.config.starterdb.globalDBName=
oracle.install.db.config.starterdb.SID=
oracle.install.db.config.starterdb.characterSet=AL32UTF8
oracle.install.db.config.starterdb.memoryOption=true
oracle.install.db.config.starterdb.memoryLimit=
oracle.install.db.config.starterdb.installExampleSchemas=false
oracle.install.db.config.starterdb.enableSecuritySettings=true
oracle.install.db.config.starterdb.password.ALL=
oracle.install.db.config.starterdb.password.SYS=
oracle.install.db.config.starterdb.password.SYSTEM=
oracle.install.db.config.starterdb.password.SYSMAN=
oracle.install.db.config.starterdb.password.DBSNMP=
oracle.install.db.config.starterdb.control=DB_CONTROL
oracle.install.db.config.starterdb.gridcontrol.gridControlServiceURL=
oracle.install.db.config.starterdb.automatedBackup.enable=false
oracle.install.db.config.starterdb.automatedBackup.osuid=
oracle.install.db.config.starterdb.automatedBackup.ospwd=
oracle.install.db.config.starterdb.storageType=
oracle.install.db.config.starterdb.fileSystemStorage.dataLocation=
oracle.install.db.config.starterdb.fileSystemStorage.recoveryLocation=
oracle.install.db.config.asm.diskGroup=
oracle.install.db.config.asm.ASMSNMPPassword=
MYORACLESUPPORT_USERNAME=
MYORACLESUPPORT_PASSWORD=
SECURITY_UPDATES_VIA_MYORACLESUPPORT=
DECLINE_SECURITY_UPDATES=true
PROXY_HOST=
PROXY_PORT=
PROXY_USER=
PROXY_PWD=
COLLECTOR_SUPPORTHUB_URL=
oracle.installer.autoupdates.option=
oracle.installer.autoupdates.downloadUpdatesLoc=
AUTOUPDATES_MYORACLESUPPORT_USERNAME=
AUTOUPDATES_MYORACLESUPPORT_PASSWORD=
```

### db_install_one.rsp
```
[GENERAL]
RESPONSEFILE_VERSION = "11.2.0"
OPERATION_TYPE = "createDatabase"
[CREATEDATABASE]
GDBNAME = "one"
SID = "one"
TEMPLATENAME = "General_Purpose.dbc"
SYSPASSWORD = "oracle"
SYSTEMPASSWORD = "oracle"
DATAFILEDESTINATION = "/oradata"
CHARACTERSET = "ZHS16GBK"
TOTALMEMORY = "4096"
[createTemplateFromDB]
SOURCEDB = "myhost:1521:orcl"
SYSDBAUSERNAME = "system"
TEMPLATENAME = "My Copy TEMPLATE"
[createCloneTemplate]
SOURCEDB = "orcl"
TEMPLATENAME = "My Clone TEMPLATE"
[DELETEDATABASE]
SOURCEDB = "orcl"
[generateScripts]
TEMPLATENAME = "New Database"
GDBNAME = "orcl11.us.oracle.com"
[CONFIGUREDATABASE]
[ADDINSTANCE]
DB_UNIQUE_NAME = "orcl11g.us.oracle.com"
NODELIST=
SYSDBAUSERNAME = "sys"
[DELETEINSTANCE]
DB_UNIQUE_NAME = "orcl11g.us.oracle.com"
INSTANCENAME = "orcl11g"
SYSDBAUSERNAME = "sys"
```
### listener.ora
```shell
vi /oracle/dbsoft/network/admin/listener.ora
  SID_LIST_LISTENER =
    (SID_LIST =
        (SID_DESC =
          (GLOBAL_DBNAME = sysMonitor)
          (ORACLE_HOME = /oracle/product/11g)
          (SID_NAME = moninfo)
        )
    )

  LISTENER =
      (DESCRIPTION =
        (ADDRESS = (PROTOCOL = TCP)(HOST = 10.10.250.87)(PORT = 1521))
      )

  ADR_BASE_LISTENER = /oracle
```
### dbca_del.rsp
```
[GENERAL]
RESPONSEFILE_VERSION = "11.2.0"
OPERATION_TYPE = "deleteDatabase"
 

[DELETEDATABASE]
SOURCEDB = "moninfo"
SYSDBAUSERNAME = "sys"
SYSDBAPASSWORD = "oracle"
```

