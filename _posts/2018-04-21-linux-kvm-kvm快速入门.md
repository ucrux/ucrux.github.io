---
author: asmbits
comments: true
date: 2018-04-21 21:14:00 +0000
layout: post
title: kvm快速入门
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- kvm
---

**安装环境:centos6**

<!-- more -->

安装epel源
```shell
rpm -Uvh  http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
```

安装kvm
```shell
yum install -y qemu-kvm
#管理工具
yum install virt-manager python-virtinst qemu-kvm-tools -y
```

虚拟机的创建
===

## 创建虚拟机硬盘
```shell
qemu-img create -f raw /opt/kvm.raw 10G
qemu-img info /opt/kvm.raw
```
## 运行虚拟机
```shell
#直接使用qemu运行
/usr/libexec/qemu-kvm

qemu-system-x86_64 -m 2048 -smp 4 -boot order=cd -hda rhel6u4.img -cdrom ./rhel-server-6.4-x86_64-dvd.iso  -enable-kvm -vnc :10 

    #use vncviewer to install guest os,and ctrl+alt+2 to switch to qemu monitor, ctrl+alt+1 to switch back
```

## 使用virsh管理
```shell
yum install libvirt libvirt-python -y

cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 tools
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6 tools

10.0.0.8    tools

#启动前添加hosts,否则会出警告
/etc/init.d/libvirtd start

#创建虚拟机
virt-install --virt-type kvm --name kvm-demo \
--ram 512 --cdrom=/opt/CentOS-6.6-x86_64-bin-DVD1.iso \
--network network=default --graphics vnc,listen=0.0.0.0 \
--noautoconsole --os-type=linux --os-variant=rhel6 --disk path=/opt/kvm.raw
```

```shell
tree
.
├── libvirt.conf
├── libvirtd.conf
├── lxc.conf
├── nwfilter
│   ├── allow-arp.xml
│   ├── allow-dhcp-server.xml
│   ├── allow-dhcp.xml
│   ├── allow-incoming-ipv4.xml
│   ├── allow-ipv4.xml
│   ├── clean-traffic.xml
│   ├── no-arp-ip-spoofing.xml
│   ├── no-arp-mac-spoofing.xml
│   ├── no-arp-spoofing.xml
│   ├── no-ip-multicast.xml
│   ├── no-ip-spoofing.xml
│   ├── no-mac-broadcast.xml
│   ├── no-mac-spoofing.xml
│   ├── no-other-l2-traffic.xml
│   ├── no-other-rarp-traffic.xml
│   ├── qemu-announce-self-rarp.xml
│   └── qemu-announce-self.xml
├── qemu
│   ├── kvm-demo.xml
│   └── networks
│       ├── autostart
│       │   └── default.xml -> ../default.xml
│       └── default.xml                 #可修改默认网络设置
└── qemu.conf

```

### virsh简用
```shell
virsh list --all        #查看所有虚拟机
 Id    Name                           State
----------------------------------------------------
 -     kvm-demo                       shut off
#查看虚拟配置
cat /etc/libvirt/qemu/kvm-demo.xml
#编辑虚拟机配置
virsh edit kvm-demo
#启动虚拟机
virsh start kvm-demo
#查看已启动的虚拟机
virsh list
 Id    Name                           State
----------------------------------------------------
 4     kvm-demo                       running
```

#### 虚拟机内存管理
**注意默认单位是KiB**
```shell
#这个需要关闭虚拟机来调整
virsh setmaxmem kvm-demo 800M
#然后就可以在这个范围内调整虚拟机的内存了
virsh setmem kvm-demo 512M
```

#### 虚拟机拷贝
```shell
virsh dumpxml kvm-demo > new.xml
cp kvm.raw new.raw
#修改new.xml
#name
#uuid
#mac
virsh define new.xml
```

#### 虚拟机硬盘操作
```shell
#改变磁盘大小
qemu-img resize new.raw +1G    #不建议使用哦
#转化磁盘格式(从raw转化为qcow2)
qemu-img convert -c -f raw -O qcow2 new.raw new.qcow2
```
1. 转化格式后 要用 virsh edit kvm-new 修改配置文件
2. 修改结束后,需要清除kvm进程,在重启启动虚拟机(virsh destroy kvm-new && virsh start kvm-new)

#### 虚拟机快照
```shell
#创建快照
qemu-img snapshot -c backup /opt/new.qcow2
#查看快照
qemu-img snapshot -l /opt/new.qcow2
Snapshot list:
ID        TAG                 VM SIZE                DATE       VM CLOCK
1         backup                    0 2017-03-20 04:16:17   00:00:00.000
#恢复快照
```

#### 虚拟机监控
```shell
yum install virt-top

virt-top
```

KVM优化
===

```
user sys        vm-user vm-sys  p-user  p-sys

L1d cache:             32K          #数据缓存
L1i cache:             32K          #指令缓存

利用cpu亲和性,减少cache miss
taskset

ksm内存共享
大页内存,默认不会被交换出去

#减少内存交换
cat /proc/sys/vm/swappiness 
60

#磁盘IO调度策略
cat /sys/block/sda/queue/scheduler 
noop anticipatory deadline [cfq]
```
