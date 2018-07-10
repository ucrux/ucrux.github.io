---
author: ucrux
comments: true
date: 2018-04-21 22:42:00 +0000
layout: post
title: linux heartbeat配置使用
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- 常用服务
---

>heartbeat 官网 http://linux-ha.org

```

              send heartbeat              | if not get 
    master ---------------------> backup  | the heartbeat
     vip                                  | from the master
  resources                               | then get the vip,resources,etc.
     etc.                                 | broadcast ARP

```

<!-- more -->

- heartbeat 高可用是服务器级别的,不是服务级别的
切换常见条件,**应用服务故障不会切换**
  - 1. 服务器宕机
  - 2. heartbeat服务本身故障
  - 3. 心跳连接故障
- heartbeat心跳连接 
  - 1. 串行电缆(首选,缺点是距离不能太远)
  - 2. 以太网线主备直连(推荐)
  - 3. 以太网线通过交换机连接(次选)
- heartbeat裂脑
  - 在指定时间内,无法相互检测到对方心跳,最*可怕*的情况就是会发生**主备双活**
  - 裂脑的原因
    - 心跳链路故障
      - 心跳物理设备故障
      - 软件防火墙原因
      - 心跳网卡配置问题
  - 防止裂脑的方法
    - 1. 冗余心跳(以太网,串口,磁盘)
    - 2. 检测到裂脑时强行关闭一个节点,如备节点发现脑裂,直接发送命令关闭主节点
    - 3. 做好裂脑监控报警,人为介入仲裁
    - 4. 启用磁盘锁,只在发生脑裂的情况下,正在使用资源的一方才对磁盘加锁
    - 5. 增加其他自动仲裁机制
- heartbeat消息类型
  - 方式 
    - 单播
    - 多播
    - 广播
  - 类型
    - 心跳消息
    - 集群转化消息
      - ip-request         主-->备,要求备机释放接管的资源
      - ip-request-resp    备-->主,通知主服务器资源已释放
    - 重传消息
      - rexmit-request     重传心跳请求

**heartbeat通过ip地址接管和ARP广播来进行故障转移**

- VIP: virtaul ip,实际上就是服务IP
- IP别名: eth0:x x为0-255
  - 添加 ifconfig eth0:0 10.0.0.254/24 up
  - 删除 ifconfig eth0:0 down
- 辅助IP: 需要通过 *ip ad* 来查看
  - 添加 ip addr add 10.0.0.254/24 broadcast 10.0.0.255 dev eth0
  - 删除 ip addr del 10.0.0.254/24 broadcast 10.0.0.255 dev eth0

- heartbeat脚步默认目录
  - 1. 启动脚本 /etc/init.d/heartbeat
  - 2. 资源目录 /etc/ha.d/resource.d/
- heartbeat常用配置文件
  - 1. ha.cf        heartbeat基本参数
  - 2. authkey      服务之间的认证
  - 3. haresource   资源配置文件


实验配置
===

## 拓扑

|   主机名    |        IP       |  用途  |
|-------------|-----------------|--------|
| heartbeat01 | 192.168.122.101 | 管理IP |
|             | 10.0.0.101      | 心跳IP |
|             | 192.168.122.103 | VIP    |
| heartbeat02 | 192.168.122.102 | 管理IP |
|             | 10.0.0.101      | 心跳IP |
|             | 192.168.122.104 | VIP    |


## 修改/etc/hosts
```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6


192.168.122.101         heartbeat01
10.0.0.101              hb-hb01
192.168.122.103         VIP-hb01

192.168.122.102         heartbeat02
10.0.0.102              hb-hb02
192.168.122.104         VIP-hb02
```

## 通过路由让10.0.0.101与10.0.0.102直连
```shell
#on 192.168.122.101
/sbin/route add -host 10.0.0.102 dev eth1 && \
echo /sbin/route add -host 10.0.0.102 dev eth1 >> /etc/rc.local

#on 192.168.122.102
/sbin/route add -host 10.0.0.101 dev eth1 && \
echo /sbin/route add -host 10.0.0.101 dev eth1 >> /etc/rc.local
```

## 安装heartbeat
```shell
rpm -Uvh  http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
yum install heartbeat -y
```
heartbeat模板配置文件
```
ll /usr/share/doc/heartbeat-3.0.4/
total 144
-rw-r--r-- 1 root root  1873 12月  3 2013 apphbd.cf
-rw-r--r-- 1 root root   645 12月  3 2013 authkeys
-rw-r--r-- 1 root root  3701 12月  3 2013 AUTHORS
-rw-r--r-- 1 root root 58752 12月  3 2013 ChangeLog
-rw-r--r-- 1 root root 17989 12月  3 2013 COPYING
-rw-r--r-- 1 root root 26532 12月  3 2013 COPYING.LGPL
-rw-r--r-- 1 root root 10502 12月  3 2013 ha.cf
-rw-r--r-- 1 root root  5905 12月  3 2013 haresources
-rw-r--r-- 1 root root  2935 12月  3 2013 README

```

## 配置heartbeat
### 拷贝模板配置文件
```shell
cd /usr/share/doc/heartbeat-3.0.4/ && \
cp ha.cf haresources authkeys /etc/ha.d/ && \
chmod 600 /etc/ha.d/authkeys        #必须是600权限
```

### ha.cf参数说明

|             参数            |              说明             |
|-----------------------------|-------------------------------|
| debugfile /var/log/ha-debug | 调试日志                      |
| logfile /var/log/ha-log     | 日志                          |
| logfacility local1          | 日志设备                      |
| keepalive 2                 | 心跳间隔时间,2秒              |
| deadtime 30                 | 心跳停止30秒后,资源漂移       |
| warntime 10                 | 10秒收不到心跳,警告计入日志   |
| initdead 120                | heartbeat启动120秒后,启动资源 |
| bcast eth1                  | 心跳广播方式,使用eth1         |
| mcast eth1 225.0.0.1 694 10 | 设置广播通信使用的端口,多播   |
| auto_failback on            | 服务自动回切                  |
| node heartbeat01            | 主节点名                      |
| node heartbeat02            | 备节点名                      |
| crm no                      | 关闭集群资源管理              |


### heartbeat各配置文件
#### authkeys
```shell
auth 1
1 sha1 47e9336850f1db6fa58bc470bc9b7810eb397f04    #随便写一个,两边一样就行
```

#### ha.cf
```shell
debugfile /var/log/ha-debug
logfile /var/log/ha-log
logfacility local1

keepalive 2
deadtime 30
warntime 10
initdead 60

mcast eth1 225.0.0.10 694 1 0

auto_failback on
node heartbeat01
node heartbeat02
crm no
```

#### haresource
```shell
#VIP
heartbeat01 IPaddr::192.168.122.103/24/eth0        #service ip01
heartbeat02 IPaddr::192.168.122.104/24/eth0        #service ip02
```

## heartbeat其命令
```shell
ls -l /usr/share/heartbeat/
total 80
-rwxr-xr-x 1 root root 21417 12月  3 2013 BasicSanityCheck
-rwxr-xr-x 1 root root  1021 12月  3 2013 ha_config
-rwxr-xr-x 1 root root  1094 12月  3 2013 ha_propagate
-rwxr-xr-x 1 root root   652 12月  3 2013 hb_addnode
-rwxr-xr-x 1 root root   652 12月  3 2013 hb_delnode
-rwxr-xr-x 1 root root   379 12月  3 2013 hb_setsite
-rwxr-xr-x 1 root root   393 12月  3 2013 hb_setweight
-rwxr-xr-x 1 root root  1133 12月  3 2013 hb_standby     #将此节点设为standby
-rwxr-xr-x 1 root root   951 12月  3 2013 hb_takeover        #接管资源
-rwxr-xr-x 1 root root  1678 12月  3 2013 mach_down
-rwxr-xr-x 1 root root  2436 12月  3 2013 req_resource
-rwxr-xr-x 1 root root 10680 12月  3 2013 ResourceManager
-rwxr-xr-x 1 root root  1518 12月  3 2013 TestHeartbeatComm
```
