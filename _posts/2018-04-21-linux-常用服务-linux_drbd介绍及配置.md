---
author: asmbits
comments: true
date: 2018-04-21 22:33:00 +0000
layout: post
title: linux drbd介绍及配置
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- 常用服务
---

- BRBD
  - 基于网络的raid1
  - 工作在文件系统之下

<!-- more -->

```
          master                                             slave
-----------------------------------------------------------------------------
       file system                                        file system 
         |    ^                                              .    ^  
         |    |                                              .    .
         V    |                                              V    .
       buffer cache                                       buffer cache
       | ^                                                         . ^
       | |                                                         . .
       | |                                                         . .
       V | | -->raw device                        raw device--> |  V .
      drbd | <--                                            <-- | drbd
           | ^ | -->  tcp/ip                        tcp/ip  --> |   | 
           | |          |                                     ^     | 
           | |          |                                     |     | 
           V |          |                                     |     V 
    disk sched          |                                     |     disk sched
           | ^          |                                     |      |
           V |          |                                     |      V
        disk drive      V                                     |     disk drive
           | ^      NIC driver                            NIC driver    |
           V |          |                                     ^         V
           disk         V                                     |        disk
                       NIC  ------------------------------>  NIC
                      
```


- 同步模式
  - 实时同步:主从都写成功才返回写成功,协议C
  - 异步同步
    - 1. 协议A,本地写成功即返回成功
    - 2. 协议B,本地写成功,并将数据发送至对端,则返回成功


实验配置
===
## 拓扑

|   主机名    |        IP       |  用途  |          |
|-------------|-----------------|--------|----------|
| heartbeat01 | 192.168.122.101 | 管理IP | 数据传输 |
|             | 10.0.0.101      | 心跳IP |          |
|             | 192.168.122.103 | VIP    |          |
| heartbeat02 | 192.168.122.102 | 管理IP | 数据传输 |
|             | 10.0.0.101      | 心跳IP |          |
|             | 192.168.122.104 | VIP    |          |



## hosts文件
```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6


192.168.122.101   heartbeat01
10.0.0.101      hb-hb01
192.168.122.103   VIP-hb01

192.168.122.102   heartbeat02
10.0.0.102      hb-hb02
192.168.122.104   VIP-hb02
```

- 其他注意事项
  - 1. 防火墙
  - 2. selinux
  - 3. 时间同步
  - 4. 添加主机路由
  - 5. 元数据分区
    - /dev/vdb2 为元数据分区,大小1-2G左右,不要格式化为文件系统,否则drbd会报错:存在文件系统

## 编译安装drbd8.4.4
```shell
#获取源码
wget http://oss.linbit.com/drbd/8.4/drbd-8.4.4.tar.gz

#安装依赖包
yum install -y kernel-devel flex gcc make kernel-headers

#编译安装
export LC_ALL=C
tar xf drbd-8.4.4.tar.gz
cd drbd-8.4.4

./configure --prefix=/app/drbd8.4.4 --with-km --with-heartbeat --sysconfdir=/etc/
#--with-km 激活内核模块

#检查kernel源文件目录
ls -ld /usr/src/kernels/$(uname -r)/

make KDIR=/usr/src/kernels/$(uname -r)/
make install

modprobe drbd

#检查mod加载
lsmod | grep drbd
drbd                  327338  0
libcrc32c               1246  1 drbd
```


## 配置drbd
### drbd.conf
```shell
#/etc/drbd.conf
global{
  usage-count no;
}

common{
  protocol C;           #同步协议,同步镜像
  
  syncer{
    rate 330M;                    #同步速度
    #verify-alg crc32c;           #验证算法
    al-extents  517;
  }

  disk{
    on-io-error detach;  #出现io错误的处理方式
    no-disk-flushes;
    no-md-flushes;
  }

  net{
    sndbuf-size 512k;
    #timeout     60;     # 6 seconds (unit = 0.1 second)
    #connect-int 10;     # 10 seconds (unit = 1 second)
    #ping-int    10;     # 10 seconds (unit = 1 second)
    #ping-timeout 5;     # 500 ms     (unit = 0.1 second)
    max-buffers      8000;
    unplug-watermark 1024;
    max-epoch-size   8000;
    #ko-count   4;
    #allow-two-primaries ;
    cram-hmac-alg   "sha1";
    shared-secret   "hdhwXes23sYEhart8t";
    after-sb-0pri   disconnect;
    after-sb-1pri   disconnect;
    after-sb-2pri   disconnect;
    rr-conflict     disconnect;
    #data-integrity-alg "md5";
    #no-tcp-cork;
  }
}

resource data{                    #drbd资源,mysqldata为资源名
  #protocol C;

  #disk{
  #  on-io-error detach;          #出现io错误的处理方式
  #}

  on heartbeat01{
    device /dev/drbd0;
    disk   /dev/vdb1;
    address 192.168.122.101:7788;
    meta-disk /dev/vdb2 [0];  #meta-dist internal;  #内部模式,不需要单独给出元数据分区
  }


  on heartbeat02{
    device /dev/drbd0;
    disk   /dev/vdb1;
    address 192.168.122.102:7788;
    meta-disk /dev/vdb2 [0];  #meta-dist internal;  #内部模式,不需要单独给出元数据分区
  }
}
```

### 激活drbd
```shell
#创建资源,同时会初始化元数据
drbdadm create-md data    #data是在drbd.conf中定义的资源

#启动drbd
mkdir -p /app/drbd8.4.4/var/run/drbd
drbdadm up data
```

*"drbdadm up data"相当于三个命令*
> drbdadm attach data
> drbdadm syncer data
> drbdadm connect data

### 检查drbd进程
```shell
cat /proc/drbd 

version: 8.4.4 (api:1/proto:86-101)
GIT-hash: 74402fecf24da8e5438171ee8c19e28627e1c98a build by root@heartbeat01, 2017-04-03 22:13:27
 0: cs:Connected ro:Secondary/Secondary ds:Inconsistent/Inconsistent C r-----
     ns:0 nr:0 dw:0 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:d oos:2663108
```

### 设置主节点
```shell
#在主节点操作,192.168.122.101
drbdadm -- --overwrite-data-of-peer primary data

#检查同步状态
##主节点
cat /proc/drbd 
version: 8.4.4 (api:1/proto:86-101)
GIT-hash: 74402fecf24da8e5438171ee8c19e28627e1c98a build by root@heartbeat01, 2017-04-03 22:13:27
 0: cs:SyncSource ro:Primary/Secondary ds:UpToDate/Inconsistent C r-----
     ns:235520 nr:0 dw:0 dr:236177 al:0 bm:14 lo:0 pe:0 ua:0 ap:0 ep:1 wo:d oos:2427588
    [>...................] sync'ed:  9.1% (2427588/2663108)K
      finish: 0:00:20 speed: 117,760 (117,760) K/sec

##备节点
cat /proc/drbd 
version: 8.4.4 (api:1/proto:86-101)
GIT-hash: 74402fecf24da8e5438171ee8c19e28627e1c98a build by root@heartbeat02, 2017-04-03 22:13:28
 0: cs:SyncTarget ro:Secondary/Primary ds:Inconsistent/UpToDate C r-----
     ns:0 nr:586752 dw:586752 dr:0 al:0 bm:35 lo:1 pe:7 ua:0 ap:0 ep:1 wo:d oos:2076356
    [===>................] sync'ed: 22.3% (2076356/2663108)K
      finish: 0:00:21 speed: 97,792 (97,792) want: 102,400 K/sec
```

>ro: 角色
>cs: 连接角色
>ds: 同步状态


### 问题解决
**角色为: Secondary/Unknown时**

备节点上操作
```shell
drbdadm secondary data
drbdadm disconnect data
drbdadm -- --discard-my-data connect data
```

主节点上
```shell
drbdadm connect data
```

### drbd文件系统的使用
```shell
#在主节点上创建文件系统
mkfs.ext4 /dev/drbd0      #只有主节点上的drbd块设备才能操作
mkdir /data
mount /dev/drbd0 /data


#如果要使用备节点文件系统,要注意同步状态
drbdadm down data
mount /dev/vda1 /data
```
