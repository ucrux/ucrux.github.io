---
author: asmbits
comments: true
date: 2018-04-21 14:24:00 +0000
layout: post
title: linux部分内核参数说明
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- kernel
---

**centos6/7 yum install kernel-doc 安装内核文档**

<!-- more -->

## net.ipv4.conf.default.rp_filter
> net.ipv4.conf.default.rp_filter=1 启用路由喝茶功能

```
rp_filter - INTEGER
        0 - No source validation.
        1 - Strict mode as defined in RFC3704 Strict Reverse Path 
            Each incoming packet is tested against the FIB and if the interface
            is not the best reverse path the packet check will fail.
            By default failed packets are discarded.
        2 - Loose mode as defined in RFC3704 Loose Reverse Path 
            Each incoming packet's source address is also tested against the FIB
            and if the source address is not reachable via any interface
            the packet check will fail.

        Current recommended practice in RFC3704 is to enable strict mode 
        to prevent IP spoofing from DDos attacks. If using asymmetric routing
        or other complicated routing, then loose mode is recommended.

        The max value from conf/{all,interface}/rp_filter is used 
        when doing source validation on the {interface}.

        Default value is 0. Note that some distributions enable it
        in startup scripts.
```

## accept_source_route
> accept_source_route=0 仅用所有IP源路由

```
accept_source_route - BOOLEAN
        Accept packets with SRR option.
        conf/all/accept_source_route must also be set to TRUE to accept packets
        with SRR option on the interface
        default TRUE (router)
                FALSE (host)
```

## sysrq
> kernel.sysrq = 0 使用sysrq组合键是了解系统目前运行情况,为安全起见设为0关闭

```
*  What is the magic SysRq key?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
It is a 'magical' key combo you can hit which the kernel will respond to
regardless of whatever else it is doing, unless it is completely locked up.

*  How do I enable the magic SysRq key?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
You need to say "yes" to 'Magic SysRq key (CONFIG_MAGIC_SYSRQ)' when
configuring the kernel. When running a kernel with SysRq compiled in,
/proc/sys/kernel/sysrq controls the functions allowed to be invoked via
the SysRq key. By default the file contains 1 which means that every
possible SysRq request is allowed (in older versions SysRq was disabled
by default, and you were required to specifically enable it at run-time
but this is not the case any more). Here is the list of possible values
in /proc/sys/kernel/sysrq:
   0 - disable sysrq completely
   1 - enable all functions of sysrq
  >1 - bitmask of allowed sysrq functions (see below for detailed function
       description):
          2 - enable control of console logging level
          4 - enable control of keyboard (SAK, unraw)
          8 - enable debugging dumps of processes etc.
         16 - enable sync command
         32 - enable remount read-only
         64 - enable signalling of processes (term, kill, oom-kill)
        128 - allow reboot/poweroff
        256 - allow nicing of all RT tasks
```

## core_uses_pid
> kernel.core_uses_pid=1 控制core文件的文件名是否添加pid作为扩展

```
The default coredump filename is "core".  By setting
core_uses_pid to 1, the coredump filename becomes core.PID.
If core_pattern does not include "%p" (default does not)
and core_uses_pid is set, then .PID will be appended to
the filename.
```

## tcp_syncookies
> net.ipv4.tcp_syncookies=1 开启SYN Cookies，当出现SYN等待队列溢出时，启用cookies来处理

```
tcp_syncookies - BOOLEAN
        Only valid when the kernel was compiled with CONFIG_SYN_COOKIES
        Send out syncookies when the syn backlog queue of a socket
        overflows. This is to prevent against the common 'SYN flood attack'
        Default: 1

        Note, that syncookies is fallback facility.
        It MUST NOT be used to help highly loaded servers to stand
        against legal connection rate. If you see SYN flood warnings
        in your logs, but investigation shows that they occur
        because of overload with legal connections, you should tune
        another parameters until this warning disappear.
        See: tcp_max_syn_backlog, tcp_synack_retries, tcp_abort_on_overflow.

        syncookies seriously violate TCP protocol, do not allow
        to use TCP extensions, can result in serious degradation
        of some services (f.e. SMTP relaying), visible not by you,
        but your clients and relays, contacting you. While you see
        SYN flood warnings in logs not being really flooded, your server
        is seriously misconfigured.

        If you want to test which effects syncookies have to your
        network connections you can set this knob to 2 to enable
        unconditionally generation of syncookies.
```

## msgmnb
> kernel.msgmnb=65536 每个消息队列的大小(单位:字节)限制

## msgmax
> kernel.msgmas=65536 整个系统最大消息队列数量限制

## shmmax
> kernel.shmmax=1092616192 单个共享内存段的大小(单位:字节)限制,计算公式1G*1024*1024*1024(字节)

```
This value can be used to query and set the run time limit
on the maximum shared memory segment size that can be created.
Shared memory segments up to 1Gb are now supported in the
kernel.  This value defaults to SHMMAX.
```

## shmall
> 所有共享内存大小(单位:页,1页 = 4Kb),计算公式16*1024*1024*1024/4KB(页) (64G)

```
This parameter sets the total amount of shared memory pages that
can be used system wide. Hence, SHMALL should always be at least
ceil(shmmax/PAGE_SIZE).

If you are not sure what the default PAGE_SIZE is on your Linux
system, you can run the following command:

# getconf PAGE_SIZE
```

## tcp_max_tw_buckets
> net.ipv4.tcp_max_tw_buckets=50000 timewait的数量,默认是4096

```
Maximal number of timewait sockets held by system simultaneously.
If this number is exceeded time-wait socket is immediately destroyed
and warning is printed. This limit exists only to prevent
simple DoS attacks, you _must_ not lower the limit artificially,
but rather increase it (probably, after increasing installed memory),
if network conditions require more than default value.
```

## tcp_sack
> net.ipv4.tcp_sack = 1 开启有选择的应答

## tcp_window_scaling
> net.ipv4.tcp_window_scaling = 1 支持更大的TCP窗口.如果TCP窗口最大超过65535(64K),必须设置该数值为1

## tcp_rmem and tcp_wmem
> net.ipv4.tcp_rmem = 4096 87380 4194304 TCP读buffer
> net.ipv4.tcp_wmem = 4096 16384 4194304 TCP写buffer 

```
vector of 3 INTEGERs: min, default, max
        min: Minimal size of receive buffer used by TCP sockets.
        It is guaranteed to each TCP socket, even under moderate memory
        pressure.
        Default: 1 page

        default: initial size of receive buffer used by TCP sockets.
        This value overrides net.core.rmem_default used by other protocols.
        Default: 87380 bytes. This value results in window of 65535 with
        default setting of tcp_adv_win_scale and tcp_app_win:0 and a bit
        less for default tcp_app_win. See below about these variables.

        max: maximal size of receive buffer allowed for automatically
        selected receiver buffers for TCP socket. This value does not override
        net.core.rmem_max.  Calling setsockopt() with SO_RCVBUF disables
        automatic tuning of that socket's receive buffer size, in which
        case this value is ignored.
        Default: between 87380B and 6MB, depending on RAM size.
```

## wmem_default and rmem_default
> net.core.wmem_default = 8388608 为TCP socket预留用于发送缓冲的内存默认值(单位:字节)
> net.core.rmem_default = 8388608 为TCP socket预留用于接收缓冲的内存默认值(单位:字节)

## rmem_max and wmem_max
> net.core.rmem_max = 16777216 The maximum receive socket buffer size in bytes.
> net.core.wmem_max = 16777216 The maximum send socket buffer size in bytes.

## netdev_max_backlog
> net.core.netdev_max_backlog = 327680 每个网络接口接收数据包的速率比内核处理这些包的速率快时,允许送到队列的数据包的最大数目

## somaxconn
> net.core.somaxconn = 65535 web应用中listen函数的backlog默认会给我们内核参数的net.core.somaxconn限制到128,而nginx定义的NGX_LISTEN_BACKLOG默认为511,所以有必要调整这个值

```
Limit of socket listen() backlog, known in userspace as SOMAXCONN.
Defaults to 128.  See also tcp_max_syn_backlog for additional tuning
for TCP sockets. 65535 max
```

## tcp_max_orphans
> net.ipv4.tcp_max_orphans = 3276800 系统中最多有多少个TCP套接字不被关联到任何一个用户文件句柄上.这个限制仅仅是为了防止简单的DoS攻击,不能过分依靠它或者人为地减小这个值,更应该增加这个值(如果增加了内存之后)

```
Maximal number of TCP sockets not attached to any user file handle,
held by system. If this number is exceeded orphaned connections are
reset immediately and warning is printed. This limit exists
only to prevent simple DoS attacks, you _must_ not rely on this
or lower the limit artificially, but rather increase it
(probably, after increasing installed memory),
if network conditions require more than default value,
and tune network services to linger and kill such states
more aggressively. Let me to remind again: each orphan eats
up to ~64K of unswappable memory
```

## tcp_max_syn_backlog
> net.ipv4.tcp_max_syn_backlog = 262144 记录的那些尚未收到客户端确认信息的连接请求的最大值.对于有128M内存的系统而言,缺省值是1024,小内存的系统则是128

## tcp_timestamps
> net.ipv4.tcp_timestamps = 0 时间戳可以避免序列号的卷绕.一个1Gbps的链路肯定会遇到以前用过的序列号.时间戳能够让内核接受这种“异常”的数据包.这里需要将其关掉

## tcp_synack_retries
> net.ipv4.tcp_synack_retries = 1 为了打开对端的连接,内核需要发送一个SYN并附带一个回应前面一个SYN的ACK.也就是所谓三次握手中的第二次握手.这个设置决定了内核放弃连接之前发送SYN+ACK包的数量

```
Number of times SYNACKs for a passive TCP connection attempt will
be retransmitted. Should not be higher than 255. Default value
is 5, which corresponds to 31seconds till the last retransmission
with the current initial RTO of 1second. With this the final timeout
for a passive TCP connection will happen after 63seconds.
```

## tcp_tw_recycle
> net.ipv4.tcp_tw_recycle = 1 开启TCP连接中time_wait sockets的快速回收

```
Enable fast recycling TIME-WAIT sockets. Default value is 0.
It should not be changed without advice/request of technical
experts.
```

## tcp_tw_reuse
> net.ipv4.tcp_tw_reuse = 1 开启TCP连接复用功能,允许将time_wait sockets重新用于新的TCP连接(主要针对time_wait连接)

```
Allow to reuse TIME-WAIT sockets for new connections when it is
safe from protocol viewpoint. Default value is 0.
It should not be changed without advice/request of technical
experts.
```

## tcp_mem
> net.ipv4.tcp_mem = 94500000 915000000 927000000 1st低于此值,TCP没有内存压力,2nd进入内存压力阶段,3rdTCP拒绝分配socket(单位:内存页)

```
vector of 3 INTEGERs: min, pressure, max
        min: below this number of pages TCP is not bothered about its
        memory appetite.

        pressure: when amount of memory allocated by TCP exceeds this number
        of pages, TCP moderates its memory consumption and enters memory
        pressure mode, which is exited when memory consumption falls
        under "min".

        max: number of pages allowed for queueing by all TCP sockets.

        Defaults are calculated at boot time from amount of available
        memory.
```

## tcp_fin_timeout
> net.ipv4.tcp_fin_timeout = 30 如果套接字由本端要求关闭,这个参数决定了它保持在FIN-WAIT-2状态的时间.对端可以出错并永远不关闭连接,甚至意外当机.缺省值是60秒. 2.2内核的通常值是180秒,你可以按这个设置,但要记住的是,即使你的机器是一个轻载的WEB服务器,也有因为大量的死套接字而内存溢出的风险,FIN- WAIT-2的危险性比FIN-WAIT-1要小,因为它最多只能吃掉1.5K内存,但是它们的生存期长些

## tcp_keepalive_time
> net.ipv4.tcp_keepalive_time = 1200 表示当keepalive起用的时候,TCP发送keepalive消息的频度(单位:秒)

```
Default: 2hours
```

## ip_local_port_range
> net.ipv4.ip_local_port_range = 2048 65535 对外连接端口范围

```
ip_local_port_range - 2 INTEGERS
        Defines the local port range that is used by TCP and UDP to
        choose the local port. The first number is the first, the
        second the last local port number.
        If possible, it is better these numbers have different parity.
        (one even and one odd values)
        The default values are 32768 and 60999 respectively.
```

## nf_conntrack_max
> net.nf_conntrack_max = 100000               centos7
> net.netfilter.nf_conntrack_max = 1048576    centos6

```
Size of connection tracking table.  Default value is
nf_conntrack_buckets value * 4.
```

