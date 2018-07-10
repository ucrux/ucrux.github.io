---
author: ucrux
comments: true
date: 2018-04-20 23:08:32 +0000
layout: post
title: linux cgroup 网络控制
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- cgroup
---

队列规则
===

tc 命令引入了一系列概念,其中我们最需要先理解的就是队列规则.它的英文名字叫做 queueing discipline ,在 tc 命令中也叫 qdisc ,或者直接简写为 qd

<!-- more -->

```shell
tc qd ls
qdisc pfifo_fast 0: dev eth0 root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
```

*qdisc 是针对网卡的,每有一个网卡就会有一个 qdisc* 

## pfifo_fast 的队列规则
- 这个 qdisc 实现了一个以数据包(package)为单位的 fifo 队列
- 实际上可以认为是实现了三个队列(bands),给每个队列定了一个优先级,以实现带优先级的排队规则
  - 每个数据包来了都根据自己的情况选择一个 band 进行排队
  - 每个 band 都是 fifo 方式处理数据包
    - 它总是先处理优先级最高的 band
    - 直到没有数据包了再处理下一个优先级的 band 
    - 直到三个都处理完,或者本次处理不完,继续等着下次处理

### 数据包选择band的规则
*priomap 1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1的含义*
* 这个字段描述了一个 priomap,可以理解为优先级位图
* 后面的 16 个不同的位,表示相关值如果为真时的数据包应该进入哪个队列
* 一共有 0 、 1 、 2 三个队列.而这个 16 位的位图标记,针对的就是我们 IP 报头中的 TOS 字段
   > 根据 IP 协议的定义我们知道:
   >    TOS 字段 8 位中的 4 位分别是用来标示最小延时,最大吞吐量,最大可靠性和最小消费四种数据包类型. 
   >    IP 协议原则上会根据这些标示的不同以不同的 QOS 对上层交付不同的服务质量.
   >    这些不同的搭配理论上一共有 16 种,这就是 priomap 所映射的优先级的概念


没有提供可供修改的参数,就是说默认的优先级分类设置不能更改,也没有提供相关限速的功能

### Bufferbloat现象

- Bufferbloat 现象最初是用来形容在一个分组交换网络上,路由器为防止丢包,往往 buffer 缓冲区都会实现的很大,但是这种过大的 fifo 缓冲区可能导致数据包 buffer 中等待时间过长而导致很多问题.再加上网络上 TCP 的拥塞控制算法的影响,以及很多商业操作系统甚至并不实现拥塞控制,导致数据传输质量抖动很大(全局同步),甚至于达到服务不可用的状态.
- 后来我们发现, Bufferbloat 这种现象比较广泛的存在在各种系统中,只要系统中使用了类似队列,缓存等机制的时候,就在某些极端状态下出现这种类似雪崩的现象


### CoDel 算法
- CoDel 采用了另外一种角度来观察队列满载的问题,其出发点并不是对队列长度进行控制,而是对队列中的数据包的驻留时间进行控制.事实上如果我们将管理方式由队列长度控制变成等待时间控制的时, bufferbloat 就可以彻底解决了,这也是更先进的 AQM 算法所用的方式

## 将队列规则改为 CoDel
```shell
[root@zorrozou-pc0 zorro]# tc qd del dev enp2s0 root
[root@zorrozou-pc0 zorro]# tc qdisc add dev enp2s0 root codel limit 100 target 4ms interval 30ms ecn
[root@zorrozou-pc0 zorro]# tc qd ls dev enp2s0
qdisc codel 8002: root refcnt 2 limit 100p target 4.0ms interval 30.0ms ecn 
[root@zorrozou-pc0 zorro]# tc -s qd ls dev enp2s0
qdisc codel 8002: root refcnt 2 limit 100p target 4.0ms interval 30.0ms ecn 
 Sent 5546 bytes 39 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0 
    count 0 lastcount 0 ldelay 0us drop_next 0us
    maxpacket 0 ecn_mark 0 drop_overlimit 0
```

使用 cgroup 限制网络流量
===

- 建立两个 cgroup:jerry,zorro
  - jerry 组中运行的网络程序限制带宽为 10mbit
  - zorro 组的网路资源占用为 20mbit
- 总带宽为 100mbit,不允许借用( ceil )网络资源
- 配置一台 centos7 的虚拟机
  - 首先,我们在这个服务器上运行一个 apache 的 http 服务
  - 并发布了一个 1G 的数据文件作为测试文件
  - 并在不限速的情况下对齐进行下载速度测试,结果为 100MBps
 
```shell
zorrozou-nb:~ zorro$ curl -O http://192.168.139.136/file
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                             Dload  Upload   Total   Spent    Left  Speed
100 1024M  100 1024M    0     0   101M      0  0:00:10  0:00:10 --:--:--  100M
```

在 centos7(192.168.139.136)上实现三个分类:

- 1. 带宽限制 10m 给 jerry 
- 2. 带宽限制 20m 给 zorro
- 3. 带宽限制 30m 用作 default 

```shell
[root@localhost Desktop]# tc qd add dev eno16777736 root handle 1: htb default 100
[root@localhost Desktop]# tc cl add dev eno16777736 parent 1: classid 1:1 htb rate 100mbit burst 20k
[root@localhost Desktop]# tc cl add dev eno16777736 parent 1:1 classid 1:10 htb rate 10mbit burst 20k
[root@localhost Desktop]# tc cl add dev eno16777736 parent 1:1 classid 1:20 htb rate 20mbit burst 20k
[root@localhost Desktop]# tc cl add dev eno16777736 parent 1:1 classid 1:100 htb rate 30mbit burst 20k

[root@localhost Desktop]# tc qd add dev eno16777736 parent 1:10 handle 10: fq_codel
[root@localhost Desktop]# tc qd add dev eno16777736 parent 1:20 handle 20: fq_codel
[root@localhost Desktop]# tc qd add dev eno16777736 parent 1:100 handle 100: fq_codel
```

建立完分类之后,由于默认情况都要走 1:100 的分类,所以限速应该是 30mbit ,验证一下:
```shell
zorrozou-nb:~ zorro$ curl -O http://192.168.139.136/file
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                             Dload  Upload   Total   Spent    Left  Speed
  0 1024M    0 3484k    0     0  3452k      0  0:05:03  0:00:01  0:05:02 3452k
```

之后将我们的 http 服务放到 zorro 组中看看效果,当然是首先建立相关 cgroup 以及相关配置：
```shell
[root@localhost Desktop]# ls /sys/fs/cgroup/net_cls/
cgroup.clone_children  cgroup.event_control  cgroup.procs  cgroup.sane_behavior  net_cls.classid  notify_on_release  release_agent  tasks
[root@localhost Desktop]# mkdir /sys/fs/cgroup/net_cls/zorro
[root@localhost Desktop]# mkdir /sys/fs/cgroup/net_cls/jerry
[root@localhost Desktop]# ls /sys/fs/cgroup/net_cls/{zorro,jerry}
/sys/fs/cgroup/net_cls/jerry:
cgroup.clone_children  cgroup.event_control  cgroup.procs  net_cls.classid  notify_on_release  tasks

/sys/fs/cgroup/net_cls/zorro:
cgroup.clone_children  cgroup.event_control  cgroup.procs  net_cls.classid  notify_on_release  tasks
```

建立完毕之后分别配置相关的 cgroup,将对应 cgroup 产生的数据包对应到相应的分类中,配置方法:
```shell
[root@localhost Desktop]# echo 0x00010100 > /sys/fs/cgroup/net_cls/net_cls.classid
[root@localhost Desktop]# echo 0x00010010 > /sys/fs/cgroup/net_cls ／ jerry/net_cls.classid
[root@localhost Desktop]# echo 0x00010020 > /sys/fs/cgroup/net_cls ／ zorro/net_cls.classid
[root@localhost Desktop]# tc fi add dev eno16777736 parent 1: protocol ip prio 1 handle 1: cgroup
```

- 这里的 tc 命令是对 filter 进行操作,这里使用了 cgroup 过滤器,来实现将 cgroup 的数据包送到 1:0 分类中.
- 对于 net_cls.classid 文件,我们一般 echo 的是一个 0xAAAABBBB 的值,AAAA 对应 class 中:前面的数字,而 BBBB 对应后面的数字
  - 如: 0x00010100 就表示这个组的数据包将被分类到 1:100 中,限速为 30mbit

之后我们把 http 服务放倒 jerry 组中看看效果:
```shell
[root@localhost Desktop]# for i in `ps ax|grep httpd|awk '{ print $1}'`;do echo $i > /sys/fs/cgroup/net_cls/jerry/tasks;done
bash: echo: write error: No such process
[root@localhost Desktop]# cat /sys/fs/cgroup/net_cls/jerry/tasks
75733
75734
75735
75736
75737
75738
75777
75778
75779
```
测试效果:
```shell
zorrozou-nb:~ zorro$ curl -O http://192.168.139.136/file
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                             Dload  Upload   Total   Spent    Left  Speed
  0 1024M    0 5118k    0     0  1162k      0  0:15:01  0:00:04  0:14:57 1162k
```


再来看看放倒 zorro 组下:
```she;;
[root@localhost Desktop]# for i in `ps ax|grep httpd|awk '{ print $1}'`;do echo $i > /sys/fs/cgroup/net_cls/zorro/tasks;done
bash: echo: write error: No such process
[root@localhost Desktop]# cat /sys/fs/cgroup/net_cls/zorro/tasks
75733
75734
75735
75736
75737
75738
75777
75778
75779
```
测试效果：
```shell
zorrozou-nb:~ zorro$ curl -O http://192.168.139.136/file
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                             Dload  Upload   Total   Spent    Left  Speed
  0 1024M    0 5586k    0     0  2334k      0  0:07:29  0:00:02  0:07:27 2334k
```

如果想要修改对于一个分类的限速,使用如下命令即可:
```shell
tc cl change dev eno16777736 parent 1: classid 1:100 htb rate 100mbit
```
