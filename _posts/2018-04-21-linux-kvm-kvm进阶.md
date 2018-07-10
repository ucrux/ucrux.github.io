---
author: asmbits
comments: true
date: 2018-04-21 21:16:00 +0000
layout: post
title: kvm进阶
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- kvm
---

使用qemu启动虚拟机
===

```
qemu-system-x86_64 -m 2048 -smp 4 -boot order=cd \
-hda rhel6u4.img -cdrom ./rhel-server-6.4-x86_64-dvd.iso \
-enable-kvm -vnc :10 

    #use vncviewer to install guest os,and ctrl+alt+2 to switch to qemu monitor, ctrl+alt+1 to switch back

#change VNC passwd
qemu-system-x86_64 -m 1024 -smp 2,maxcpus=8,sockets=1,cores=1,threads=2 \
-boot c -drive file=rhel6u4.img,if=virtio,format=raw -enable-kvm \
-mem-path /dev/hugepages -mem-prealloc \
-net nic -net user,tftp=/VMs,hostfwd=tcp::5022-:22 \
-vnc :10,password -monitor stdio

###ballnoon  
qemu-system-x86_64 -m 1024 -smp 2,maxcpus=8,sockets=1,cores=1,threads=2 \
-boot c -drive file=rhel6u4.img,if=virtio,format=raw \
-enable-kvm -net nic -net user,hostfwd=tcp::5022-:22 \
-vnc :90 -monitor stdio \
-balloon virtio
# -ballnoon virtio

#in qemu monitor
info balloon #get guest balloon info
balloon num  #set mem of guest to num MB
```

<!-- more -->

CPU亲和性
===
```shell
###### use CGROUP to adjust affinity, but i dont know how now #######
vi /boot/grub/grub.conf
    title CentOS (4.4.12)
        root (hd0,0)
        kernel /vmlinuz-4.4.12 ro root=/dev/mapper/rootvg-lvroot rd_NO_LUKS  KEYBOARDTYPE=pc KEYTABLE=us LANG=en_US.UTF-8 rd_LVM_LV=rootvg/lvswap rd_NO_MD SYSFONT=latarcyrheb-sun16 crashkernel=auto rd_LVM_LV=rootvg/lvroot rd_NO_DM rhgb quiet isolcpus=2,3
        initrd /initramfs-4.4.12.img
    #add isolcpus=2,3 隔离 cpu2 cpu3. and reboot
ps -eLo psr | grep 0 | wc -l     #check count of threads in every cpu
ps -eLo psr | grep 1 | wc -l
ps -eLo psr | grep 2 | wc -l
ps -eLo psr | grep 3 | wc -l

# 只有内核线程在cpu3上运行
ps -eLo ruser,pid,ppid,lwp,psr,args | awk '{if($5==3) print $0}'
root        21     2    21   3 [watchdog/3]
root        22     2    22   3 [migration/3]
root        23     2    23   3 [ksoftirqd/3]
root        24     2    24   3 [kworker/3:0]
root        25     2    25   3 [kworker/3:0H]
root        42     2    42   3 [kworker/3:1]
root      1032     2  1032   3 [kworker/3:1H]

# 检查虚拟机的cpu线程ID
ps -eLo ruser,pid,ppid,lwp,psr,args | grep qemu | grep -v grep
root      1510     1  1510   0 qemu-system-x86_64 -m 2048 -smp 2,maxcpus=8,sockets=1,cores=1,threads=2 -boot c -hda deb8u2.img -enable-kvm -vnc :10 -daemonize
root      1510     1  1512   1 qemu-system-x86_64 -m 2048 -smp 2,maxcpus=8,sockets=1,cores=1,threads=2 -boot c -hda deb8u2.img -enable-kvm -vnc :10 -daemonize
root      1510     1  1513   0 qemu-system-x86_64 -m 2048 -smp 2,maxcpus=8,sockets=1,cores=1,threads=2 -boot c -hda deb8u2.img -enable-kvm -vnc :10 -daemonize
root      1510     1  1515   1 qemu-system-x86_64 -m 2048 -smp 2,maxcpus=8,sockets=1,cores=1,threads=2 -boot c -hda deb8u2.img -enable-kvm -vnc :10 -daemonize

#Affinity new cpus 
# 0010 cpu1, 0001 cpu0
taskset -p 0x04 1510      #0100     cpu2     
taskset -p 0x04 1512      #0100     cpu2
taskset -p 0x08 1513      #1000     cpu3
taskset -p 0x08 1515      #1000     cpu3

#再次查看虚拟机cpu线程所在的cpu
ps -eLo ruser,pid,ppid,lwp,psr,args | awk '{if($5==3) print $0}'
root        21     2    21   3 [watchdog/3]
root        22     2    22   3 [migration/3]
root        23     2    23   3 [ksoftirqd/3]
root        24     2    24   3 [kworker/3:0]
root        25     2    25   3 [kworker/3:0H]
root        42     2    42   3 [kworker/3:1]
root      1032     2  1032   3 [kworker/3:1H]
root      1510     1  1513   3 qemu-system-x86_64 -m 2048 -smp 2,maxcpus=8,sockets=1,cores=1,threads=2 -boot c -hda deb8u2.img -enable-kvm -vnc :10 -daemonize
root      1510     1  1515   3 qemu-system-x86_64 -m 2048 -smp 2,maxcpus=8,sockets=1,cores=1,threads=2 -boot c -hda deb8u2.img -enable-kvm -vnc :10 -daemonize

ps -eLo ruser,pid,ppid,lwp,psr,args | awk '{if($5==2) print $0}'
root        16     2    16   2 [watchdog/2]
root        17     2    17   2 [migration/2]
root        18     2    18   2 [ksoftirqd/2]
root        19     2    19   2 [kworker/2:0]
root        20     2    20   2 [kworker/2:0H]
root        41     2    41   2 [kworker/2:1]
root      1012     2  1012   2 [kworker/2:1H]
root      1510     1  1510   2 qemu-system-x86_64 -m 2048 -smp 2,maxcpus=8,sockets=1,cores=1,threads=2 -boot c -hda deb8u2.img -enable-kvm -vnc :10 -daemonize
root      1510     1  1512   2 qemu-system-x86_64 -m 2048 -smp 2,maxcpus=8,sockets=1,cores=1,threads=2 -boot c -hda deb8u2.img -enable-kvm -vnc :10 -daemonize
```

页大小(page size,使用大页)
===

```shell
# 查看系统默认页大小
getconf PAGESIZE
# 挂载大页设备
mount -t hugetlbfs hugetlbfs /dev/hugepages
# 调整系统大页数量
sysctl vm.nr_hugepages=1024
# 使用如下命令启动虚拟机,虚拟机将使用大页
qemu-system-x86_64 -m 1024 -smp 2,maxcpus=8,sockets=1,cores=1,threads=2 \
-boot c -hda deb8u2.img -enable-kvm -vnc :10 -daemonize \
-mem-path /dev/hugepages -mem-prealloc
#if use huge page, the mem CANNOT swap out, and it can ballooning inceasing
```

虚拟机迁移
===
```
###share storage
#source host
qemu-system-x86_64 -m 1024 -smp 2,maxcpus=8,sockets=1,cores=1,threads=2 \
-boot c -drive file=rhel6u4.img,if=virtio,format=raw -enable-kvm \
-net nic -net user,hostfwd=tcp::5022-:22 -vnc :90 \
-monitor stdio -balloon virtio

#dest host
qemu-system-x86_64 -m 1024 -smp 2,maxcpus=8,sockets=1,cores=1,threads=2 \
-boot c -drive file=rhel6u4.img,if=virtio,format=raw -enable-kvm \
-net nic -net user,hostfwd=tcp::5022-:22 -vnc :90 \
-monitor stdio -balloon virtio -incoming tcp:0:6666
# need -incoming tcp:0:6666; 0:from any host; 6666:tcp port

#source host operationg in qemu monitor
migrate tcp:dest_host:6666 
#if using backing_file, need migrate -i tcp:dest_host:6666
#if no share storage, migrate -b tcp:dest_host:6666
```

