---
author: ucrux
comments: true
date: 2018-04-21 14:30:00 +0000
layout: post
title: macos xhyve轻量级虚拟化的使用
image: /assets/images/blog_back.jpg
categories:
- macos
tags:
- 软件使用
---


如何使用xhyve
===

## 介绍

FreeBSD 下的虚拟技术 bhyve (The BSD Hypervisor) 是去年1月份正式发布的,包含在了 FreeBSD 10.0 发行版中. xhyve 是基于 bhyve 的 Mac OS X 移植版本. 也就是说我们想在 Mac 上运行 Linux 的话除了 VirtualBox, VMware Fusion 外,现在有了第三种选择. 

## 安装xhyve

### 安装要求

- OS X 10.10.3 Yosemite or later
- a 2010 or later Mac (i.e. a CPU that supports EPT)

### 安装

If you have homebrew, then simply:

```shell
brew update
brew install --HEAD xhyve
```

> The --HEAD in the brew command ensures that you always get the latest changes, even if the homebrew database is not yet updated. If for any reason you don't want that simply do brew install xhyve .

if not then:
Building

```shell
git clone https://github.com/mist64/xhyve
cd xhyve
make
```

> The resulting binary will be in build/xhyve

### 打印usage

```shell
xhyve -h
```

## 使用xhyve安装ubuntu

新建一个 ubuntu 目录用来存放所有和 ubuntu 虚拟机相关的东西,下载 ubuntu-14.04.2-server-amd64.iso，并把 iso 里面的两个系统启动需要的文件 vmlinuz 和 initrd.gz 拷贝出来：

```shell
mkdir ~/VMs/ubuntu16.04 -pv && mkdir ~/VMs/ISO -pv

cd ~/VMs/ISO && \
curl -O http://releases.ubuntu.com/16.04.3/ubuntu-16.04.3-server-amd64.iso
```

因为现在的 centos 镜像是hybrid file system的(可以直接dd 到u盘烧录的),而 OS X 的hdiutil 不支持直接挂载,所以我们需要一点小小的magic
```
dd if=/dev/zero bs=2k count=1 of=/tmp/ubuntu.iso && \
dd if=ubuntu-16.04.3-server-amd64.iso bs=2k skip=1 >> /tmp/ubuntu.iso

hdiutil attach /tmp/ubuntu.iso && \
cp /Volumes/Ubuntu-Server\ 16/install/vmlinuz . && \
cp /Volumes/Ubuntu-Server\ 16/install/initrd.gz .
```

创建一个 10GB 大小的硬盘文件当作 ubuntu 虚拟机的硬盘:

```
dd if=/dev/zero of=ubuntu16.04_root.vhd bs=1g count=10
```

创建虚拟机安装脚本

```shell
#!/bin/sh

KERNEL="${HOME}/VMs/ubuntu16.04/vmlinuz"
INITRD="${HOME}/VMs/ubuntu16.04/initrd.gz"
CMDLINE="earlyprintk=serial console=ttyS0 acpi=off"

MEM="-m 1G"
#SMP="-c 2"
NET="-s 2:0,virtio-net"
IMG_CD="-s 3,ahci-cd,${HOME}/VMs/ISO/ubuntu-16.04.3-server-amd64.iso"
IMG_HDD="-s 4,virtio-blk,${HOME}/VMs/ubuntu16.04/ubuntu16.04_root.vhd"
PCI_DEV="-s 0:0,hostbridge -s 31,lpc"
LPC_DEV="-l com1,stdio"

xhyve $MEM $SMP $PCI_DEV $LPC_DEV $NET $IMG_CD $IMG_HDD -f kexec,$KERNEL,$INITRD,"$CMDLINE"
```

**启动这个文件需要 sudo 权限(添加桥接网卡需要root权限)**

```shell
sudo sh ubuntu16.04_install.sh
```

这时候会看到 ubuntu 的标准文本格式的安装程序,安装过程中唯一要注意的是硬盘分区不要选择 LVM 分区.

>本例中分区: /dev/vda1 /boot ; /dev/vda2 /

**还有一个要注意的地方,安装完毕后,这时候选择 Go Back,因为我们要到 Execute a shell 命令行界面把里面的内核文件拷贝出来留作以后启动用**

选择 Execute a shell 后转到目标目录,知道虚拟机的 IP 地址后用 nc 把虚拟机和外面的世界（Mac）连起来传输文件:

```shell
cd /target/
tar c boot | nc -l -p 9000

# if centos
dhclient eth0
cd /mnt/sysimage/boot/
python -m SimplyHTTPServer
```

在 Mac 上接受文件:

```shell
cd ${HOME}/VMs/ubuntu16.04
nc 192.168.64.3 9000 | tar x    #192.168.64.3为虚拟机IP

# if centos
curl -O http://192.168.64.2:8000/vmlinuz-3.10.0-229.el7.x86_64
curl -O http://192.168.64.2:8000/initramfs-3.10.0-229.el7.x86_64.img
```

虚拟机启动脚本

```shell
#!/bin/sh

KERNEL="${HOME}/VMs/ubuntu16.04/boot/vmlinuz-4.4.0-87-generic"
INITRD="${HOME}/VMs/ubuntu16.04/boot/initrd.img-4.4.0-87-generic"
#/dev/vdaX 和 /(根分区) 分区一致
CMDLINE="earlyprintk=serial console=ttyS0 acpi=offi root=/dev/vda2 ro"

MEM="-m 1G"
SMP="-c 2"
NET="-s 2:0,virtio-net"
#IMG_CD="-s 3,ahci-cd,${HOME}/VMs/ISO/ubuntu-16.04.3-server-amd64.iso"
IMG_HDD="-s 4,virtio-blk,${HOME}/VMs/ubuntu16.04/ubuntu16.04_root.vhd"
PCI_DEV="-s 0:0,hostbridge -s 31,lpc"
LPC_DEV="-l com1,stdio"

xhyve $MEM $SMP $PCI_DEV $LPC_DEV $NET $IMG_CD $IMG_HDD -f kexec,$KERNEL,$INITRD,"$CMDLINE"
```

## 参考

<p><a href="https://github.com/mist64/xhyve">项目地址</a></p>
<p><a href="http://www.pagetable.com/?p=831">虚拟机安装参考</a></p>

## 附录
### homebrew 使用
#### 安装
```shell
cd ~

mkdir extapp && curl -L https://github.com/Homebrew/brew/tarball/master | tar xz --strip 1 -C extapp
```

**now enjoy it**

#### 安装其他软件包

```shell
cd extapp

bin/brew install tmux

vi ~/.tmux.conf
set-option -g mouse on
```

##### 通过proxy升级
```
ALL_PROXY=socks5://127.0.0.1:60000 bin/brew update
ALL_PROXY=socks5://127.0.0.1:60000 bin/brew upgrade
```

### centos 安装脚本
```shell
#!/bin/sh

KERNEL="${HOME}/vms/centos7x64_1503_base/boot/vmlinuz"
INITRD="${HOME}/vms/centos7x64_1503_base/boot/initrd.img"
CMDLINE="earlyprintk=serial console=ttyS0 acpi=off"

MEM="-m 1G"
#SMP="-c 2"
NET="-s 2:0,virtio-net"
IMG_CD="-s 3,ahci-cd,${HOME}/vms/iso/CentOS-7-x86_64-DVD-1503-01.iso"
IMG_HDD="-s 4,virtio-blk,${HOME}/vms/centos7x64_1503_base/vhd/centos7x64_1503_rootpv.vhd"
PCI_DEV="-s 0:0,hostbridge -s 31,lpc"
LPC_DEV="-l com1,stdio"

${HOME}/extapp/bin/xhyve $MEM $SMP $PCI_DEV $LPC_DEV $NET $IMG_CD $IMG_HDD -f kexec,$KERNEL,$INITRD,"$CMDLINE"
```

### centos 启动脚本
```shell
#!/bin/sh

KERNEL="${HOME}/vms/centos7x64_1503_base/boot/vmlinuz-3.10.0-229.el7.x86_64"
INITRD="${HOME}/vms/centos7x64_1503_base/boot/initramfs-3.10.0-229.el7.x86_64.img"
CMDLINE="earlyprintk=serial console=ttyS0 acpi=off root=/dev/vda3 ro"

MEM="-m 1G"
SMP="-c 2"
NET="-s 2:0,virtio-net"
IMG_CD="-s 3,ahci-cd,${HOME}/vms/iso/CentOS-7-x86_64-DVD-1503-01.iso"
IMG_HDD="-s 4,virtio-blk,${HOME}/vms/centos7x64_1503_base/vhd/centos7x64_1503_rootpv.vhd"
PCI_DEV="-s 0:0,hostbridge -s 31,lpc"
LPC_DEV="-l com1,stdio"

#UUID="-U cac5fd90-e6aa-4277-b8bd-26bed71fcb09"

${HOME}/extapp/bin/xhyve $MEM $SMP $PCI_DEV $LPC_DEV $NET $IMG_CD $IMG_HDD $UUID -f kexec,$KERNEL,$INITRD,"$CMDLINE"
```


