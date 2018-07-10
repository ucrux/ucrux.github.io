---
author: ucrux
comments: true
date: 2018-04-21 22:44:00 +0000
layout: post
title: linux keepalive配置使用
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- 常用服务
---

ARP协议
=== 
**局域网环境工作**

- address resolution protocol,地址解析协议
- 获取IP所对应主机的mac地址(计算通信最终是靠MAC地址)

<!-- more -->

## ARP协议工作原理
```
10.1.1.1 ----> access 10.1.1.2

10.1.1.1           10.1.1.2
check arp cache    /
   |              /   响应(单播)
   | no cached   /     ARP广播
   V            /
发送ARP广播    /
询问:         /
谁是10.1.1.2
```

**vlan隔离广播风暴**

```shell
arp -a       #查询命令
```


## ARP在高可用环境中的问题
高可用服务器切换时要考虑ARP缓存问题
**发送ARP广播通知其他服务器更改ARP缓存**
```shell
#gw
/sbin/arping -I eth0 -c 3 -s 10.0.0.30 10.0.0.2
/sbin/arping -U -I eth0 10.0.0.30
```

LVS(linux virtual server)
===

- CIP : client IP
- DIP : director IP
- VIP : virtaul IP
- RIP : real server IP
- LVS模式
  - 1.NAT(Network Address Translation)改写目的ip为RIP(不改目的MAC)
    - RIP的网关要是LVS的DIP
      - 请求的包问(DNAT)和响应的报文(SNAT),通过调度器地址重写,然后转发给内部的服务器
支持对端口的转换
    - 由于数据包来回都经过调度器,因此要开启内核转发**net.ipv4.ip_forward=1**
  - 2.TUN(Tunneling)
    - in:增加一个IP头
    - out:NULL
    - 其他类似DR
    - **可以跨机房,不局限于局域网**
  - 3.DR(Direct Routing)单臂路由模式
    - real server 需要做两个操作
      - lo绑VIP **解包的时候会发现目的地址是VIP,如果lo上不绑VIP,包会被丢弃**
      - 抑制ARP
    - real server and LVS must in the same LAN
    - 目的端口无法改变,即:VIP端口和RIP端口必须一致
    - 改写请求报文的目标MAC地址,将请求发给真实服务器
    - 真实服务器**直接**将返回发送给请求服务器
  - 4.FULLNAT(full Network Address Translation)
    - in:源,目的全改
    - out:源,目的全改
    - 其他类似NAT
    - **DNAT和SNAT可以是不同的LVS**
- LVS调度算法
  - 固定调度算法:rr,wrr,dh,sh
  - 动态调度算法:wlc,lc,lblc,lblcr,SED,NQ(后两种官方站点没提到)

| 调度算法 |             说明             |
|----------|------------------------------|
| rr       | 轮循调度                     |
| wrr      | 加权轮循调度                 |
| dh       | 目的地址哈希调度             |
| sh       | 源地址哈希调度               |
| wlc      | 加权最小连接数调度           |
| lc       | 最小连接数调度               |
| lblc     | 基于地址的最小连接数调度     |
| lblcr    | 基于地址带重复最小连接数调度 |
| SED      | 最短期望延迟调度             |
| NQ       | 最少队列调度                 |

*适用于TCP,UDP协议的应用*

LVS install
===
## 确认ipvs没安装
```
lsmod | grep ip_vs
```
## install lvs(above centos6.x)
```
yum install -y libnl* popt* kernel-devel
ln -s /usr/src/kernels/$(uname -r)/ /usr/src/linux
wget http://www.linuxvirtualserver.org/software/kernel-2.6/ipvsadm-1.26.tar.gz
tar zxf ipvsadm-1.26.tar.gz
cd ipvsadm-1.26
make
make install
lsmod | grep ip_vs
modprobe ip_vs
```

## 使用ipvs
### LVS server
```
ifconfig eth0:0 10.0.0.30/24    #vip

ipvsadm  -C                     #clean all configure

ipvsadm --set 30 5 60           #set timeout tcp tcpfin udp

#add virtual service
#-s scheduler method
#-p session hold,多少秒内只找这台机器
ipvsadm -A -t 10.0.0.30:3306  -s rr -p 20

#add real server
#-g DR mode
#-w 权重
ipvsadm -a -t 10.0.0.30:3306 -r 10.0.0.10:3306 -g -w 1 
ipvsadm -a -t 10.0.0.30:3306 -r 10.0.0.11:3306 -g -w 1

#check
ipvsadm -L -n
ipvsadm -L -n --stats --sort

#delete virtual server
ipvsadm -D -t 10.0.0.30:3306
ipvsadm -d -t 10.0.0.30:3306 -r 10.0.0.10:3306
```

### Real Server
```
##绑定VIP
ifconfig lo:0 10.0.0.30/32 up

##抑制ARP

#应答模式
#回答接了网线的网口的目的地址的ARP请求
echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore

#回应限制
#对查询目标使用最适当的本地地址
echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
```

## keepalived
**VRRP:Virtaul Router Redundancy Protocol,虚拟路由冗余协议**

* 解决静态路由的单点故障
* 通过竞选机制将路由任务交给某个VRRP路由器
* VRRP通信通过IP多播
* 由master多播,backup接收,如backup接不到包,超时backup就接管

>healthcheck:check real server
>failover:take over VIP

### keepalived install
```
#down load software
wget http://www.keepalived.org/software/keepalived-1.1.19.tar.gz

#kernel module
ln -s /usr/src/kernels/$(uname -r)/ /usr/src/linux

yum install -y openssl-devel popt*

tar xvf keepalived-1.1.19.tar.gz
cd keepalived-1.1.19

./configure
#Use IPVS Framework            : Yes
#IPVS sync daemon support      : Yes
#Use VRRP Framework            : Yes

make && make install

#copy some files for manage easily
cp /usr/local/etc/rc.d/init.d/keepalived /etc/init.d/keepalived
cp /usr/local/etc/sysconfig/keepalived /etc/sysconfig/
mkdir /etc/keepalived -p
cp /usr/local/etc/keepalived/keepalived.conf /etc/keepalived/
cp /usr/local/sbin/keepalived /usr/bin/

#start and stop keepalived
/etc/init.d/keepalived start
/etc/init.d/keepalived stop
```


### keepalived log
**/var/log/messages**

#### 指定文件接收keepalived log
```
vi /etc/sysconfig/keepalived
#change 
KEEPALIVED_OPTIONS="-D"
#to
KEEPALIVED_OPTIONS="-D -d -S 0"

vi /etc/rsyslog.conf
#add
local0.*      /var/log/keepalived.log

/etc/init.d/rsyslog restart
```

#### keepalived and LVS
>add below ti keepalived's conf file

```
virtaul_server 10.0.0.10 80 {
  delay_loop 6
  lb_algo wrr
  lb_kind DR
  net_mask 255.255.255.0
  persistence_timeout 300
  protocol TCP
#ipvsadm -A -t 10.0.0.10:80 -s wrr -p 300
  real_server 10.0.0.20 80 {
    weight 1
    TCP_CHECK {
      connect_timeout 8
      nb_get_retry 3
      delay_before_retry 3
      connect_port 90
    }
  }
  real_server 10.0.0.21 80 {
    weight 1
    TCP_CHECK {
      connect_timeout 8
      nb_get_retry 3
      delay_before_retry 3
      connect_port 90
    }
  }
#ipvsadm -a -t 10.0.0.10:80 -r 10.0.0.20:80 -g -w 1
#ipvsadm -a -t 10.0.0.10:80 -r 10.0.0.21:80 -g -w 1
}
```

#### 常用命令
```
ipvsadm -Ln --stats
ipvsadm -Lnc
ipvsadm -Ln --thresholds
ipvsadm -Ln --timeout
```

### keepalived MISC健康检查说明
```shell
#MISC healthchecker, run a program
MISC_CHECK
{
    # External system script or program
    misc_path <STRING>|<QUOTED-STRING>
    # Script execution timeout
    misc_timeout <INT>

    # If set, exit code from healthchecker is used
    # to dynamically adjust the weight as follows:
    #   exit status 0: svc check success, weight
    #     unchanged.
    #   exit status 1: svc check failed.
    #   exit status 2-255: svc check success, weight
    #     changed to 2 less than exit status.
    #   (for example: exit status of 255 would set
    #     weight to 253)
    misc_dynamic
}
```
