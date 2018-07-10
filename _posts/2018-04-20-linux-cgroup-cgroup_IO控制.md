---
author: ucrux
comments: true
date: 2018-04-20 22:55:32 +0000
layout: post
title: linux cgroup IO控制
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- cgroup
---

- 1. VFS 层:虚拟文件系统层.由于内核要跟多种文件系统打交道,而每一种文件系统所实现的数据结构和相关方法都可能不尽相同,所以,内核抽象了这一层,专门用来适配各种文件系统,并对外提供统一操作接口
- 2. 文件系统层:不同的文件系统实现自己的操作过程,提供自己特有的特征
- 3. 页缓存层
- 4. 通用块层:由于绝大多数情况的 io 操作是跟块设备打交道,所以 Linux 在此提供了一个类似 vfs 层的块设备操作抽象层.下层对接各种不同属性的块设备,对上提供统一的 Block IO 请求标准
- 5. IO调度层:因为绝大多数的块设备都是类似磁盘这样的设备,所以有必要根据这类设备的特点以及应用的不同特点来设置一些不同的调度算法和队列.以便在不同的应用环境下有针对性的提高磁盘的读写效率,这里就是大名鼎鼎的 Linux 电梯所起作用的地方.针对机械硬盘的各种调度方法就是在这实现的
- 6. 块设备驱动层:驱动层对外提供相对比较高级的设备操作接口,往往是 C 语言的,而下层对接设备本身的操作方法和规范
- 7. 块设备层:这层就是具体的物理设备了,定义了各种设备操作方法和规范

<!-- more -->

对IO的限速主要在**IO调度层**和**通用块层**实现

IO调度层
===
这一层也主要是针对用途最广泛的机械硬盘结构而设计的
## 调度算法

- 1. noop 就是空操作调度算法,也就是没有任何调度操作,并不对 io 请求进行排序,仅仅做适当的 io 合并的一个 fifo 队列
- 2. cfq 完全公平队列调度,试图给所有进程提供一个完全公平的 IO 操作环境
    它为每个进程创建一个同步 IO 调度队列,
    并默认以时间片和请求数限定的方式分配 IO 资源,
    以此保证每个进程的 IO 资源占用是公平的,
    cfq 还实现了针对进程级别的优先级调度

**cgroup blkio 的权重比例限制就是基于 cfq 调度器实现的,如果你要使用权重比例分配,请先确定对应的块设备的 IO 调度算法是 cfq**


```shell
cat /sys/block/sda/queue/scheduler 
noop deadline [cfq] 
echo cfq > /sys/block/sda/queue/scheduler
```

- 3. deadline 最终期限调度.
    实现了四个队列,其中两个分别处理正常 read 和 write,按扇区号排序,进行正常 io 的合并处理以提高吞吐量.
    因为 IO 请求可能会集中在某些磁盘位置,这样会导致新来的请求一直被合并,于是可能会有其他磁盘位置的 io 请求被饿死.
    于是实现了另外两个处理超时 read 和 write 的队列,按请求创建时间排序,如果有超时的请求出现,就放进这两个队列
    调度算法保证超时(达到最终期限时间)的队列中的请求会优先被处理,防止请求被饿死.
    *由于 deadline 的特点,无疑在这里无法区分进程,也就不能实现针对进程的 io 资源控制*


通用块设备层
===
- 一般 io
  - 一个正常的文件 io,需要经过 vfs -> buffer\page cache -> 文件系统 -> 通用块设备层 -> IO 调度层 -> 块设备驱动 -> 硬件设备这所有几个层次
- Direct io
  - VFS 之后跳过 buffer\page cache 层，直接从文件系统层进行操作
- Sync IO & write-through
  - 这种方式写的数据要等待存储写入返回才能成功返回,但是,写的数据仍然是要在 cache 中写入的,这样其他一般 IO 的程度仍然可以使用 cache 机制加速 IO 操作
  - 在执行 write 操作的时候,让 cache 和存储上的数据一致
- write-back
  - 将目前在 cache 中还没写回存储的脏数据写回到存储,内核自己找时间再去写回到存储


**内核在处理 write-back 的阶段,由于没有相关 page cache 中的 inode 是属于那个 cgroup 的信息记录,所以所有的 page cache 的回写只能放到 cgroup 的 root 组中进行限制,而不能在其他 cgroup 中进行限制,因为 root 组的 cgroup 一般是不做限制的,所以就相当于目前的 cgroup 的 blkio 对 buffered IO 是没有限速支持的**


blkio配置
===
## 权重比例配置
创建两个 cgroup 组,一个叫 test1 ,一个叫 test2 
让这两个组的进程在对 /dev/dm-0 设备号为 253:0 的这个磁盘进行读写的时候按权重比例进行 io 资源的分配

```shell
ls /cgroup/blkio/

mkdir /cgroup/blkio/test1 
mkdir /cgroup/blkio/test2

ls /cgroup/blkio/test{1,2}
```

1. blkio.weight     
2. blkio.weight_device 可以针对设备单独进行限制

设置 test1 和 test2 使用任何设备的 io 权重比例都是 1:2 
```shell
echo 100 > /cgroup/blkio/test1/blkio.weight
echo 200 > /cgroup/blkio/test2/blkio.weight
```

**测试脚本**
```
#!/bin/bash

testfile1=/home/test1
testfile2=/home/test2

if [ -e $testfile1 ]
then
    rm -rf $testfile1
fi

if [ -e $testfile2 ]
then
    rm -rf $testfile2
fi

sync
echo 3 > /proc/sys/vm/drop_caches

cgexec -g blkio:test1 dd if=/dev/zero of=$testfile1 oflag=direct bs=1M count=1023 &

cgexec -g blkio:test2 dd if=/dev/zero of=$testfile2 oflag=direct bs=1M count=1023 &
```

**限制效果**
```
iotop

Total DISK READ: 3.93 K/s | Total DISK WRITE: 131.52 M/s
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>COMMAND 
  11204 be/4 root        0.00 B/s   45.15 M/s  0.00 % 99.99 % dd if=/dev/zero of=/home/test1 oflag=direct bs=1M count=1023
  11205 be/4 root        3.93 K/s   86.37 M/s  0.00 % 97.94 % dd if=/dev/zero of=/home/test2 oflag=direct bs=1M count=1023
```

设置针对 dm-0 (lvroot) 的权重 
```shell
echo "253:0 400" > /cgroup/blkio/test1/blkio.weight_device
echo "253:0 200" > /cgroup/blkio/test2/blkio.weight_device
```

## 读写带宽和 IOPS 限制
针对读写带宽和 iops 的限制都是绝对值限制,所以不用两个 cgroup 做对比了.设置 test1 的写带宽速度为 1M/s
### 读写带宽限制
```shell
echo "253:0 1048567" > /cgroup/blkio/test1/blkio.throttle.write_bps_device
```

**测试过程**
```
sync
echo 3 > /proc/sys/vm/drop_caches
cgexec -g blkio:test1 dd if=/dev/zero of=/home/test oflag=direct count=1024 bs=1M
```

**测试结果**
```
Total DISK READ: 0.00 B/s | Total DISK WRITE: 1004.97 K/s
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
  11247 be/4 root        0.00 B/s 1004.97 K/s  0.00 % 98.11 % dd if=/dev/zero of=/home/test oflag=direct count=1024 bs=1M
```

### IOPS限制
```
echo "253:0  20" > /cgroup/blkio/test1/blkio.throttle.write_iops_device

rm /home/test
sync
echo 3 > /proc/sys/vm/drop_caches
cgexec -g blkio:test1 dd if=/dev/zero of=/home/test oflag=direct count=1024 bs=1M
```


blkio其他设置
===

## 针对权重比例限制的相关文件

- blkio.leaf_weight[_device]
  - 其意义等同于 blkio.weight[_device]
  - 主要表示当本 cgroup 中有子 cgroup 的时候
  - 本 cgroup 的进程和子 cgroup 中的进程所分配的资源比例是怎么样的

举个例子说吧，假设有一组 cgroups 的关系是这样的：
```
root
       /    |   \
      A     B    leaf
     400   200   200
```
leaf 就表示 root 组下的进程所占 io 资源的比例
此时 A 组中的进程可以占用的比例为: 400 / (400+200+200) ＊ 100% ＝ 50%
B 为: 200 / (400+200+200) ＊ 100% ＝ 25%
而 root 下的进程为: 200 / (400+200+200) ＊ 100% ＝ 25%

- blkio.time
  - 统计相关设备的分配给本组的 io 处理时间,单位为 ms. 权重就是依据此时间比例进行分配的
- blkio.sectors
  - 统计本 cgroup 对设备的读写扇区个数
- blkio.io_service_bytes
  - 统计本 cgroup 对设备的读写字节个数
- blkio.io_serviced
  - 统计本 cgroup 对设备的读写操作个数
- blkio.io_service_time
  - 统计本 cgroup 对设备的各种操作时间,时间单位是 ns
- blkio.io_wait_time
  - 统计本 cgroup 对设备的各种操作的等待时间,时间单位是 ns
- blkio.io_merged
  - 统计本 cgroup 对设备的各种操作的合并处理次数
- blkio.io_queued
  - 统计本 cgroup 对设备的各种操作的当前正在排队的请求个数
- blkio.*_recursive
  - 这一堆文件是相对应的不带_recursive 的文件的递归显示版本
  - 所谓递归的意思就是,它会显示出包括本 cgroup 在内的衍生 cgroup 的所有信息的总和

## 针对带宽和 iops 限制的相关文件

- blkio.throttle.io_serviced
  - 统计本 cgroup 对设备的读写操作个数
- blkio.throttle.io_service_bytes
  - 统计本 cgroup 对设备的读写字节个数
- blkio.reset_stats
  - 对本文件写入一个 int 可以对以上所有文件的值置零,重新开始累计
