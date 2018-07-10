---
author: asmbits
comments: true
date: 2018-04-20 23:08:32 +0000
layout: post
title: linux cgroup 内存控制
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- cgroup
---

1. 内存管理需要将内存区分为活跃的(Active)和不活跃的(Inactive),再加上一个进程使用的用户空间内存映射包括文件影射(file)和匿名影射(anon),所以就包括了 Active (anon), Inactive (anon), Active (file)和 Inactive (file)

2. 匿名影射主要是诸如进程使用 malloc 和 mmap 的 MAP_ANONYMOUS 的方式申请的内存,而文件影射就是使用 mmap 影射的文件系统上的文件,这种文件系统上的文件既包括普通的文件,也包括临时文件系统(tmpfs)

3. Sys V 的 IPC 和 POSIX 的 IPC (IPC 是进程间通信机制,在这里主要指共享内存,信号量数组和消息队列)都是通过文件影射方式体现在用户空间内存中的

4. 这两种影射的内存都会被算成进程的 RSS ,但是也一样会被显示在 cache 的内存计数中,在相关 cgroup 的另一项统计中,共享内存的使用和文件缓存(file cache)也都会被算成是 cgroup 中的 cache 使用的总量

<!-- more -->

```shell
cat /cgroup/memory/memory.stat 

cache 1588879360
rss 6422528
mapped_file 6565888
pgpgin 3405600
pgpgout 3016633
swap 0
inactive_anon 200704
active_anon 6385664
inactive_file 1574322176
active_file 14315520
unevictable 0
hierarchical_memory_limit 9223372036854775807
hierarchical_memsw_limit 9223372036854775807
total_cache 1588879360
total_rss 6422528
total_mapped_file 6565888
total_pgpgin 3405600
total_pgpgout 3016633
total_swap 0
total_inactive_anon 200704
total_active_anon 6385664
total_inactive_file 1574322176
total_active_file 14315520
total_unevictable 0
```

**swap**
1. 如果是硬盘上的文件,就不用 swap 交换出去,只要写回脏数据,保持数据一致之后清除
2. 能交换出去内存应该主要包括有 Inactive (anon) 这部分内存.内核也将共享内存作为计数统计进了 Inactive ( anon )中去了(是的，共享内存也可以被 Swap)
3. 如果内存被 mlock 标记加锁了,则也不会交换,这是对内存加 mlock 锁的唯一作用

*swap 是内部设备和外部设备的数据拷贝,那么加一个缓存就显得很有必要,这个缓存就是 swapcache ,在 memory.stat 文件中, swapcache 是跟 anon page 被一起记录到 rss 中的,但是并不包含共享内存.另外再说明一下, HugePages 也是不会交换的*


配置内存限制
===

在/etc/cgconfig.conf中添加
```shell
group one {
    cpu {
    }

    cpuset {
    }

    memory {
    }
}
```

在/etc/gcrules.conf中添加
```shell
one     cpu,cpuset.cpuacct,memory   one
```

重起服务
```shell
/etc/init.d/cgconfig restart
/etc/init.d/cgred restart
```

## cgroup内存限制说明
### 内存限制参数

1. memory.memsw.limit_in_bytes:内存＋ swap 空间使用的总量限制
2. memory.limit_in_bytes：内存使用量限制
 
**如果你决定在 cgroup 中关闭 swap 功能,可以把两个文件的内容设置为同样的值即可**

### OOM控制
memory.oom_control:内存超限之后的oom行为控制
```shell
cat /cgroup/memory/one/memory.oom_control 
oom_kill_disable 0              #默认为 0 表示打开 oom killer,就是说当内存超限时会触发干掉进程.
                                #如果设置为1 表示关闭 oom killer,此时内存超限不会触发内核杀掉进程.
                                #而是将进程夯住( hang ／ sleep ),实际上内核中就是将进程设置为 D 状态.
                                #并且将相关进程放到一个叫做 OOM-waitqueue 的队列中.
                                #这时的进程可以 kill 杀掉
                                #如果你想继续让这些进程执行,可以选择这样几个方法:
                                #   1. 增加内存,让进程有内存可以继续申请
                                #   2. 杀掉一些进程,让本组内有内存可用
                                #   3. 把一些进程移到别的 cgroup 中,让本 cgroup 内有内存可用
                                #   4. 删除一些 tmpfs 的文件,就是占用内存的文件,比如共享内存或者其它会占用内存的文件
                                #说白了就是,此时只有当cgroup中有更多内存可以用了,在 OOM-waitqueue 队列中被挂起的进程就可以继续运行了

under_oom 0                     #这个值只是用来看的,它表示当前的 cgroup 的状态是不是已经 oom 了.如果是,这个值将显示为 1 

```

### 内存资源审计

1. memory.memsw.usage_in_bytes:当前 cgroup 的内存＋ swap 的使用量
2. memory.usage_in_bytes:当前 cgroup 的内存使用量
3. memory.max_usage_in_bytes:cgroup 最大的内存＋ swap 的使用量
4. memory.memsw.max_usage_in_bytes:cgroup 的最大内存使用量
