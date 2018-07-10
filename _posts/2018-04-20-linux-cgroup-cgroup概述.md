---
author: ucrux
comments: true
date: 2018-04-20 22:40:32 +0000
layout: post
title: linux cgroup概述
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- cgroup
---

* Control Groups的缩写
* 是Linux内核提供的一种可以限制,记录,隔离进程组 (process groups) 所使用的物力资源 (如 cpu memory i/o 等等) 的机制

<!-- more -->

CGroup功能及组成
===

*CGroup支持的文件种类*

|        文件名        | R/W |                      用途                      |
|----------------------|-----|------------------------------------------------|
| Release_agent        | RW  | 删除分组时执行的命令,这个文件只存在于根分组    |
| Notify_on_release    | RW  | 设置是否执行 release_agent.为 1 时执行         |
| Tasks                | RW  | 属于分组的线程 TID 列表                        |
| Cgroup.procs         | R   | 属于分组的进程PID列表.仅包括多线程leader的 TID |
| Cgroup.event_control | RW  | 监视状态变化和分组删除事件的配置文件           |

- 相关概念
  - 1. TASK: TASK就是系统的一个进程
  - 2. control group: 控制族群就是一组按照某种标准划分的进程,Cgroups 中的资源控制都是以控制族群为单位实现
  - 3. Hierarchy(层级): 既一棵控制族群树,子节点可继承父控制族群的特定的属性
  - 4. subsystem: 一个子系统就是一个资源控制器,比如 cpu 子系统就是控制 cpu 时间分配的一个控制器.子系统必须附加（attach）到一个层级上才能起作用,一个子系统附加到某个层级以后,这个层级上的所有控制族群都受到这个子系统的控制
- 相关关系
  - 1. 每次在系统中创建新层级时,该系统中的所有任务都是那个层级的默认 cgroup（我们称之为 root cgroup,此 cgroup 在创建层级时自动创建,后面在该层级中创建的 cgroup 都是此 cgroup 的后代）的初始成员
  - 2. 一个子系统最多只能附加到一个层级
  - 3. 一个层级可以附加多个子系统
  - 4. 一个任务可以是多个 cgroup 的成员,但是这些 cgroup 必须在不同的层级
  - 5. 系统中的进程(任务)创建子进程(任务)时,该子任务自动成为其父进程所在 cgroup 的成员.然后可根据需要将该子任务移动到不同的 cgroup 中,但开始时它总是继承其父任务的 cgroup

*CGroup 层级关系显示,CPU 和 Memory 两个子系统有自己独立的层级系统,而又通过 Task Group 取得关联关系*

```
        Hierarchy                                     Hierarchy
        (cpu)                                         (mem)
      /        \          ---- Task group -----       /       \
   cgroup1    cgroup2    /       |     |       \-- cgroup5   cgroup6
             /       \  /      task1  task2                   /    \
          cgroup3   cgroup4                              cgroup7  cgroup8

```


CGroup部署及应用实例
===

>实验环境 centos6.8x64

## 服务启动
```shell
#安装
yum install -y libcgroup

#启动服务
/etc/init.d/cgconfig start

#查看
ls /cgroup/
blkio  cpuacct  devices  memory
cpu    cpuset   freezer  net_cls
```

cgroup cpu资源实验
===
## 实验脚本
```shell
#!/bin/bash 

x=0

while [ True ]
do
    x=$x+1
done

```

## 用top查看cpu占用
```
PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND                    
1519 root      20   0  105m 2148 1508 R 100.0  0.1   0:21.25 bash 
```

**此进程占用cpu 100%**

## 使用cgroup控制这个进程的cpu资源
```shell
mkdir -p /cgroup/cpu/foo            #新建一个控制族群foo

#初始未对cpu user资源作限制
cat /cgroup/cpu/foo/cpu.cfs_quota_us 
-1

#cpu user 可使用的时间片
cat /cgroup/cpu/foo/cpu.cfs_period_us 
100000

#现在限制其使用为50%
echo 50000 > /cgroup/cpu/foo/cpu.cfs_quota_us # 相对于cpu.cfs_period_us的100000是50%

#将进程添加到控制族群foo中去
echo 1519 > /cgroup/cpu/foo/tasks
```

## 控制后的效果
```
PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
1519 root      20   0  107m 3808 1508 R 50.2  0.1  13:22.93 bash 
```

**cpu使用只有50%**


cgroup mem资源实验
===
## 实验脚本
```shell
#!/bin/bash
x="a"
while [ True ];do
    x=$x$x
done;
```

看见内存逐渐上升
```
PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
1519 root      20   0  490m 142m 1512 R 50.2  3.7  16:05.57 bash
```

## 使用cgroup控制该进程的内存使用
```shell
mkdir -p /cgroup/memory/foo
echo 1048576 > /cgroup/memory/foo/memory.limit_in_bytes     #分配1M的内存给这个控制族群
echo <pid> > /cgroup/memory/foo/tasks
```

**可以看到进程直接被杀死**

>可以通过配置关掉cgroups oom kill进程,通过memory.oom_control来实现(oom_kill_disable 1),但是尽管进程不会被直接杀死,但进程也进入了休眠状态,无法继续执行,仍无法服务


cgroup io资源实验
===
## 实验命令
```
dd if=/dev/vda of=/dev/null &
```

未作控制前的IO使用情况
```
1609 be/4 root      328.95 M/s    0.00 B/s  0.00 % 62.44 % dd if=/dev/vda of=/dev/nul
```

## 使用cgroup控制io资源
```shell
mkdir -p /cgroup/blkio/foo

#查看设备主次设备号
ls -l /dev/vda
brw-rw---- 1 root disk 252, 0 Mar 31 10:16 /dev/vda

#252:0对应主设备号和副设备号,可以通过ls -l /dev/vda查看
echo "252:0     1048576" > /cgroup/blkio/foo/blkio.throttle.read_bps_device

echo <pid> > /cgroup/blkio/tasks
```

## 控制后的效果
```
1619 be/4 root     1135.01 K/s    0.00 B/s  0.00 % 99.99 % dd if=/dev/vda of=/dev/null
```


- linux组织CPU资源的组织
  - 1. 按照核心数组织
  - 2. 按照使用率来组织

**分时**

```shell
grep -i hz  /boot/config-2.6.32-642.15.1.el6.x86_64 
CONFIG_NO_HZ=y
# CONFIG_HZ_100 is not set
# CONFIG_HZ_250 is not set
# CONFIG_HZ_300 is not set
CONFIG_HZ_1000=y
CONFIG_HZ=1000                      #1000之间中断为一个时间片
CONFIG_MACHZ_WDT=m
```

## cgroup分配cpu资源

**man 5 cgrules.conf**

```shell
cat /etc/cgrules.conf 
# /etc/cgrules.conf
#The format of this file is described in cgrules.conf(5)
#manual page.
#
# Example:
#<user>     <controllers>   <destination>
#@student   cpu,memory  usergroup/student/
#peter      cpu     test1/
#%      memory      test2/
# End of file
one     cpu,cpuset,cpuacct      one         #用户名  cpu的时间非配,cpu的核心分配,cpu记账   cg组名
richar  cpu,cpuset,cpuacct      richar  


#重起服务
/etc/init.d/cgred restart
```

**man 5 cgconfig.conf**

```shell
cat /etc/cgconfig.conf 
#
#  Copyright IBM Corporation. 2007
#
#  Authors: Balbir Singh <balbir@linux.vnet.ibm.com>
#  This program is free software; you can redistribute it and/or modify it
#  under the terms of version 2.1 of the GNU Lesser General Public License
#  as published by the Free Software Foundation.
#
#  This program is distributed in the hope that it would be useful, but
#  WITHOUT ANY WARRANTY; without even the implied warranty of
#  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
#
# See man cgconfig.conf for further details.
#
# By default, mount all controllers to /cgroup/<controller>

mount {
    cpuset  = /cgroup/cpuset;
    cpu = /cgroup/cpu;
    cpuacct = /cgroup/cpuacct;
    memory  = /cgroup/memory;
    devices = /cgroup/devices;
    freezer = /cgroup/freezer;
    net_cls = /cgroup/net_cls;
    blkio   = /cgroup/blkio;
}


group one {
    cpu {
        #cpu.shares = 240000;
        cpu.cfs_quota_us = "-1";
    }

    cpuset {
        #cpuset.cpus = "0-1,3";
        #cpuset.cpus = "2,3";
        #cpuset.cpus = "0-3";
    }
}

group richar {
    cpu {
        #cpu.shares = 80000;
        cpu.cfs_quota_us = "1800000";
    }

    cpuset {
        #cpuset.cpus = "1,3";
        #cpuset.cpus = "0-1";
    }
}

#重器服务
/etc/init.d/cgconfig restart
```

```shell
#检查
ls /cgroup/cpu
one             richar


ls /cgroup/cpuset/
one             richar
```

## 测试代码
```c
/*
    筛选质数
*/
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define NUM 48              //开启48个线程
#define START 100010001     //计算质数的起始值
#define END 100020000       //计算的结束值

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
static int count = 0;

void *prime(void *p)
{
    int n, i, flag;

    while (1) {
            if (pthread_mutex_lock(&mutex) != 0) {
                    perror("pthread_mutex_lock()");
                    pthread_exit(NULL);
            }
            while (count == 0) {
                    if (pthread_cond_wait(&cond, &mutex) != 0) {
                            perror("pthread_cond_wait()");
                            pthread_exit(NULL);
                    }
            }
            if (count == -1) {
                    if (pthread_mutex_unlock(&mutex) != 0) {
                            perror("pthread_mutex_unlock()");
                            pthread_exit(NULL);
                    }
                    break;
            }
            n = count;
            count = 0;
            if (pthread_cond_broadcast(&cond) != 0) {
                    perror("pthread_cond_broadcast()");
                    pthread_exit(NULL);
            }
            if (pthread_mutex_unlock(&mutex) != 0) {
                    perror("pthread_mutex_unlock()");
                    pthread_exit(NULL);
            }
            flag = 1;
            for (i=2;i<n/2;i++) {
                    if (n%i == 0) {
                            flag = 0;
                            break;
                    }
            }
            if (flag == 1) {
                    printf("%d is a prime form %d!\n", n, pthread_self());
            }
    }
    pthread_exit(NULL);
}


int main(void)
{
    pthread_t tid[NUM];
    int ret, i;

    for (i=0;i<NUM;i++) {
            ret = pthread_create(&tid[i], NULL, prime, NULL);
            if (ret != 0) {
                    perror("pthread_create()");
                    exit(1);
            }
    }

    for (i=START;i<END;i+=2) {
            if (pthread_mutex_lock(&mutex) != 0) {
                    perror("pthread_mutex_lock()");
                    pthread_exit(NULL);
            }
            while (count != 0) {
                    if (pthread_cond_wait(&cond, &mutex) != 0) {
                            perror("pthread_cond_wait()");
                            pthread_exit(NULL);
                    }
            }
            count = i;
            if (pthread_cond_broadcast(&cond) != 0) {
                    perror("pthread_cond_broadcast()");
                    pthread_exit(NULL);
            }
            if (pthread_mutex_unlock(&mutex) != 0) {
                    perror("pthread_mutex_unlock()");
                    pthread_exit(NULL);
            }
    }

    if (pthread_mutex_lock(&mutex) != 0) {
            perror("pthread_mutex_lock()");
            pthread_exit(NULL);
    }
    while (count != 0) {
            if (pthread_cond_wait(&cond, &mutex) != 0) {
                    perror("pthread_cond_wait()");
                    pthread_exit(NULL);
            }
    }
    count = -1;
    if (pthread_cond_broadcast(&cond) != 0) {
            perror("pthread_cond_broadcast()");
            pthread_exit(NULL);
    }
    if (pthread_mutex_unlock(&mutex) != 0) {
            perror("pthread_mutex_unlock()");
            pthread_exit(NULL);
    }

    for (i=0;i<NUM;i++) {
            ret = pthread_join(tid[i], NULL);
            if (ret != 0) {

                    perror("pthread_join()");
                    exit(1);
            }
    }

    exit(0);
}

```

## 编译测试代码
```shell
gcc test_code.c -lpthread
```

测试 
=== 
## cpu亲和性测试
```shell
#one用户
cpuset.cpus = '0-1';
#cpuset.mems = '0-1';       #可访问两路CPU,且两路cpu的内存都能访问
```
### cpu路数查看
```shell
numactl -H
available: 1 nodes (0)
node 0 cpus: 0 1 2 3
node 0 size: 4095 MB
node 0 free: 3584 MB
node distances:
node   0 
  0:  10 
```

## cpu分时
```shell
cat /cgroup/cpu/one/cpu.cfs_period_us 
100000

#让其使用所有cpu的25%
cpu.cfs_period_us = 18000       # 25% * 核心数 * /cgroup/cpu/one/cpu.cfs_period_us     
```

## cpu权重
```shell
#one
    cpu.shares = 6000;

#richar
    cpu.shares = 18000;

#安装shares的值的全重进行cpu时间的分配
```
