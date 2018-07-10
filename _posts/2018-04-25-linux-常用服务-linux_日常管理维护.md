---
author: ucrux
comments: true
date: 2018-04-25 21:18:00 +0000
layout: post
title: linux 日常管理维护
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- 常用服务
---


常用授时中心
===

- 微软：time.windows.com
- 台湾：asia.pool.ntp.org
- 中科院：210.72.145.44
- 网通：219.158.14.130

<!-- more -->

获取机器序列号
===
```shell
dmidecode
```

使用certbot生成免费ssl证书
===

## IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/ucrux.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/ucrux.com/privkey.pem
   Your cert will expire on 2018-03-02. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot-auto
   again. To non-interactively renew *all* of your certificates, run
   "certbot-auto renew"
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.

```shell
wget https://dl.eff.org/certbot-auto
chmod +x certbot-auto
mv certbot-auto /usr/local/bin/

# 此处要注意 python2.6 和 python2.7
# 签发证书
# 如果使用 --standalone 签发证书,那么每次更新证书必须保证 443 端口不被占用
certbot-auto certonly --standalone -d ucrux.com -d www.ucrux.com -d jmp.ucrux.com -d log.ucrux.com

# 自动更新
## 测试自动更新
certbot-auto renew --dry-run

vi /etc/crontab
11 23 */2 * * /usr/local/bin/certbot-auto renew --quiet
```

LVM条带化
===
```shell
    # 安装时请在命令行中建立PV、VG、LV
    # 记得要单独建立/boot文件系统

    ## pv
    pvcreate /dev/sda2
    pvcreate /dev/sdb
    pvcreate /dev/sdc

    ##vg
    vgcreate -l 1024 -p 1024 -s 8M rootvg /dev/sda2 /dev/sdb /dev/sdc

    ##lv
    lvcreate -L 5G -i 3 -I 32 -n lvroot rootvg
    lvcreate -L 2G -i 3 -I 32 -n lvswap rootvg
    lvcreate -l 100%free -i 3 -I 32 -n lvhome rootvg

    ##show
    lvs -v --segments
```

bond网卡
===

```shell
# 修改network
vi /etc/sysconfig/network
VLAN=yes

# 建立bond网卡配置文件
vi /etc/sysconfig/network-script/bond0
DEVICE="bond0"
BONDING_OPTS="mode=0 miimon=100"
NM_CONTROLLED="no"
ONBOOT=no
BOOTPROTO=none

# 编辑从属网卡配置文件
vi /etc/sysconfig/network-script/ifcfg-eth0
# Intel Corporation 82580 Gigabit Network Connection
DEVICE=eth0
HWADDR=90:E2:BA:27:57:72
#IPADDR=10.10.30.250
#NETMASK=255.255.255.0
BOOTPROTO="none"
IPV6INIT="no"
NM_CONTROLLED="no"
ONBOOT="yes"
TYPE="Ethernet"
SLAVE="yes"
MASTER="bond0"

vi /etc/sysconfig/network-script/ifcfg-eth4
# Intel Corporation 82580 Gigabit Network Connection
DEVICE=eth4   
HWADDR=90:E2:BA:27:57:88
#IPADDR=192.168.2.251
#NETMASK=255.255.255.0
BOOTPROTO="none"
IPV6INIT="no"
NM_CONTROLLED="no"
ONBOOT="yes"
TYPE="Ethernet"
SLAVE="yes"
MASTER="bond0"

# 在/etc/modprobe.conf加入
vi /etc/modprobe.conf
alias bond0 bonding
options bond0 miimon=100 mode=0 （mode 0 是双工，1是互备 ）

# 激活bond网卡
ifenslave bond0 eth0 eth4

# 添加vlan和vlan网卡配置文件
vconfig add bond0 503
vi /etc/sysconfig/network-scripts/ifcfg-bond0.503
DEVICE="bond0.503"
NM_CONTROLLED="no"
ONBOOT=no
BOOTPROTO=static
IPADDR=10.10.250.250
NETMASK=255.255.255.0

# 在rc.loacl中添加开机启动
vi /etc/rc.local
ifenslave bond0 eth0 eth4
vconfig add bond0 503
ifup bond0.503
# 注意：一台机器只能有一个网关
```

iptables配置
===
## 查看本机的iptables设置
```shell
iptables -nL --line-number
iptables -L -n
    Chain INPUT (policy ACCEPT)
    target     prot opt source               destination         
    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           state RELATED,ESTABLISHED
    ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0           
    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           state NEW tcp dpt:22
    REJECT     all  --  0.0.0.0/0            0.0.0.0/0           reject-with icmp-host-prohibited

    Chain FORWARD (policy ACCEPT)
    target     prot opt source               destination         
    REJECT     all  --  0.0.0.0/0            0.0.0.0/0           reject-with icmp-host-prohibited

    Chain OUTPUT (policy ACCEPT)
    target     prot opt source               destination
```
## 清除规则
```shell
iptables -F     #清除预设表filter中的所有规则链的规则
iptables -X     #清除预设表filter中使用者自定链中的规则
```
## 保存设置
```shell
/etc/rc.d/init.d/iptables save
```
## 初始化配置
```shell
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP
```
以上规则丢弃所有的包，也就是说网络不能用
## 打开ssh登录
```shell
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT
```
## 允许icmp包通过,也就是允许ping
```shell
iptables -A OUTPUT -p icmp -j ACCEPT    #(OUTPUT设置成DROP的话)
iptables -A INPUT -p icmp -j ACCEPT     #(INPUT设置成DROP的话)
```
## 允许loopback!(不然会导致DNS无法正常关闭等问题)
```shell
IPTABLES -A INPUT -i lo -p all -j ACCEPT         #(如果是INPUT DROP)
IPTABLES -A OUTPUT -o lo -p all -j ACCEPT     #(如果是OUTPUT DROP)
```
## 允许RouteDtoC的ip转送
```shell
vi /etc/sysctl.conf
    ...
    net.ipv4.ip_forward = 1   # 修改为1
    ...
```
### 允许iptables forward中 eth0 到eth1
```shell
iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
```
### 利用NAT将 172.168.185.0 网段转发到10.10.10.0网段
```shell
iptables -t nat -A POSTROUTING -s 172.168.185.0/24 -d 10.10.10.0/24 -j SNAT --to 10.10.10.1       #需要地址伪装时才用
```

存储在线管理
===

## Fibre Channel驱动
Linux系统上Fibre Channel驱动用户空间接口位于/sys/class文件夹下面.以下条目中，
H代表主机，B代表bus号，T代表target，L代表lun id，R代表对端端口号.

- Transport： /sys/class/fc_transport/targetH:B:T/
  - port_id – 存储端口的24位交换机端口ID
  - node_name – 存储端口的64位node name
  - port_name – 存储端口的64位port name
- Remote Port：/sys/class/fc_remote_ports/rport-H:B-R/
  - port_id – 存储端口的24位交换机端口ID
  - node_name – 存储端口的64位node name
  - port_name – 存储端口的64位port name
  - dev_loss_tmo – 链路故障等待时间.故障链路不再处理任何新的IO.默认dev_loss_tmo值视具体HBA卡而定，Qlogic默认是35秒，Emulex默认是 30秒.HBA卡自带驱动可以覆盖这个参数值.dev_loss_tmo最大值600秒，如果dev_loss_tmo值小于0或者大于600，HBA自带超时值生效.
  - fast_io_fail_tmo – IO故障等待时间.链路波动情况，IO尝试多长时间.
- Host：/sys/class/fc_host/hostH
  - port_id – HBA端口的 24位交换机端口ID.
  - issues_lip – 重置HBA端口，重新尝试发现存储端口.
- Persistent命名:操作系统通过路径发送IO到存储，Linux系统SCSI磁盘路径有以下部分组成：
  - HBA卡的PCI标示符
  - HBA卡的管道号
  - 存储端SCSI target地址
  - LUN(Logical Unit Number) 号
- SCSI磁盘路径在Linux上有3中表现方式：
  - /dev/sd目录；
  - 通过major:minor号；
  - /dev/disk/by-path，该目录是 /dev/sd设备的软连接.Fibre Channel磁盘路径示范如下：
   - pci-0000:42:00.0-fc-0x2014d4ae52a263ef-lun-18 -> ../../sdt
   - PCI标示  chanel号     存储WWN号      LUN_ID

## 磁盘路径确认
### 1. WWID
根据SCSI标准,每个SCSI磁盘都有一个WWID.类似于网卡的MAC地址,要求是独一无二.通过WWID标示SCSI磁盘就可以保证磁盘路径永久不变,Linux系统上/dev/disk/by-id目录包含每个SCSI磁盘WWID访问路径.
实例:<br>
scsi-3600508b400105e210000900000490000 -> ../../sda<br>
提示:Linux自带的device-mapper-multipath工具就是通过WWID来探测SCSI磁盘路径,可以将同一设备多条路径合并,并在/dev/mapper/下面创建新的设备路径.通过multipath –l可以看到WWID与 磁盘路径,Host:Channel:Target:Lun与/dev/sd以及major:minor对应关系.
```
    mpathw (36d4ae52000a2638f0000096a54d09393) dm-38 DELL,MD36xxf
    size=500G features='3 queue_if_no_path pg_init_retries 50' hwhandler='1 rdac' wp=rw
    |-+- policy='round-robin 0' prio=14 status=active
    | `- 1:0:0:18 sdt  65:48  active ready running
    `-+- policy='round-robin 0' prio=9 status=enabled
     `- 2:0:0:18 sdar 66:176 active ready running
```

### 2. UUID
UUID是有文件系统在创建时候生成的,用来标记文件系统,类似WWID一样也是独一无二的.因此使用UUID来标示SCSI磁盘,也能保证路径是永久不变的.<br>
Linux上/dev/disk/by-uuid可以看到每个已经创建文件系统的磁盘设备以及与/dev/sd之间的映射关键.<br>
注意：Linux自带的md和LVM工具也会在SCSI磁盘上面写入UUID信息.

### 3. UDEV
UDEV是Linux提供的一种让用户对设备进行自定义命名的机制.可以通过UDEV将WWID/UUID信息跟磁盘路径映射起来,这样也可以保证设备路径永久不变.

## 磁盘删除
在删除磁盘之前,建议先备份好数据,将内存脏数据写入磁盘,然后再删除磁盘所有关联路径.对于使用多路径软件的磁盘,需要同时删除多路径设备和每个路径磁盘.删除磁盘建议在系统空闲时操作,内存脏数据写入磁盘会增加系统负载,可以通过vmstat 1 100观察系统负载.如果满足一下条件之一,则不建议进行删除操作:<br>

- vmstat 100次输出结果超过10次以上的free内存小于系统内存的5%.
- vmstat结果的si和so列不为空，代表系统正在进行swaping，将内存数据写入磁盘.

### 磁盘删除操作步骤如下:

1. 关闭使用磁盘的进程，备份磁盘数据.可以通过fuser命令查看正在访问某个磁盘的进程.
2. blockdev –flushfs 将脏数据写入磁盘.这一步骤对于裸设备情况尤为重要，因为裸设备无法通过umount或者vgreduce将脏数据写入磁盘.
3. umount卸载基于待删除磁盘的文件系统
4. md和LVM中删除磁盘.LVM可以使用vgreduce从卷组移除改磁盘，然后使用pvremove从磁盘删除LVM元数据.
5. 如果磁盘使用多路径软件，通过mulitpath –l查看磁盘所有路径，然后通过multipath –f删除磁盘.（如果使用powerpath多路径软件，请参考powerpath操作手册）
6. 删除应用程序或者脚本中的磁盘路径引用.
7. 使用命令echo 1 > /sys/block/device-name/device/delete删除磁盘，device-name以sd开头，比如sda、sdb;或者使用命令echo 1 > /sys/class/scsi_device/h:c:t:l/device/delete删除磁盘，h代表HBA卡号，c代表HBA卡 channel，t代表SCSI target ID，l代表LUN ID.h:c:t:l这些信息可以通过lsscsi，scsi_id，multipath –l，ls –l /dev/disk/by-*方式查看.
8. 直接删除磁盘文件.

### 磁盘路径删除

- 当系统使用多路径软件时候，可以在线删除一条路径而不影响业务使用.操作步骤如下：
  - 1. 在应用程序或脚本中删除磁盘路径引用.
  - 2. 使用命令echo offline > /sys/block/sda/device/state将磁盘路径offline.多路径软件将会使用剩余路径处理IO.
  - 3. 使用命令echo 1 > /sys/block/device-name/delete命令从SCSI子系统删除磁盘路径，device-name通常以sd开头，比如sda、sdb.

## 添加新磁盘或者磁盘路径
添加新磁盘或者磁盘路径, 系统可能会自动分配老磁盘使用的名称给新磁盘, 比如/dev/sd,major:minoe和/dev/disk/by-path.因此在添加之前,请确认应用程序和脚本已经删除老磁盘的引用.

- 添加新磁盘或者磁盘路径步骤如下:
  - 1.完成交换机和存储配置，记录号存储端口的WWPN.
  - 2.使用下面命令在系统上重新扫描磁盘设备.
    - echo “h c t l” > /sys/class/scsi_host/hosth/scan
      - h代表HBA号
      - c代表HBA卡channel
      - t代表SCSI target ID
      - l代表LUN ID
    - 也可以使用 命令echo “scsi add-single-device 0 0 0 0” > /proc/scsi/scsi完成扫描.

> a. 某些HBA卡在存储分配完成后,需要通过issue_lip才能发现新增加设备,具体参考“存储链路扫描”.(如果需要issue_lip，建议停止IO操作)<br>
> b. 新分配LUN在操作系统没有显示,可以通过sg_luns命令(来自sg3_utils包)重新获取阵列LUN列表.<br>
> c. c:t:l信息可以使用命令grep '存储SP端口WWPN' /sys/class/fc_transport/*/[node_name|port_name]获取；也可以通过lsscsi、scsi_id、 multipath –l或者ls –l /dev/disk/by-*方法获取.

  - 3.使用多路径软件multipath(或者其他多路径软件)确认磁盘路径添加正常.

## 存储链路扫描
Linux操作系统提供多种存储链路重置操作.存储链路重置通常用于多路径设备添加或者删除,这是一种破坏性操作,将导致IO操作超时.请谨慎使用这类型操作,另外操作时请注意以下事项:

1. 操作前请确认链路没有新的IO，脏数据已经写回存储.
2. 在系统内存资源使用紧张情况下，不建议进行链路删除操作.系统内存使用情况可以通过vmstat 1 100命令评估.

- 以下命令可用于存储链路重新扫描
  - echo "1" > /sys/class/fc_host/host/issue_lip
    - issue_lip重置HBA链路，重新扫描整个链路并配置SCSI target.该操作是一种异步操作类型，具体完成时间需要参考/vag/log/message日志.Linux操作系统自带的lpfc和qla2xxx 驱动支持issue_lip命令.
  - echo "- - -" > /sys/class/scsi_host/hostH/scan
  - rmmod 驱动/ modprobe 驱动 删除/重新加载HBA卡驱动.


挂载windows共享文件夹
===
```shell
mount -t cifs -o username="DGDSLR\administrator",password="x10i_dslr*.*2013",rw,dir_mode=0777,file_mode=0777  //10.10.10.10/software$ /mnt/windows_share/
```

热插拔scsi设备
===
## 添加scsi设备
```shell
echo "scsi add-single-device w x y z" > /proc/scsi/scsi

#为使该命令正常运行，必须指定正确的参数值 w、x、y 和 z，如下所示：
#w 是主机适配器标识，第一个适配器为零（0）
#x 是主机适配器上的 SCSI 通道，第一个通道为零（0）
#y 是设备的 SCSI 标识
#z 是 LUN 号，第一个 LUN 为零（0）
```
## 删除scsi设备
```shell
echo "scsi remove-single-device w x y z" > /proc/scsi/scsi
```

最小化安装CentOS6 VMware-tools依赖包
===
```shell
yum groupinstall "Perl Support" -y
yum install gcc gcc-c++ automake make kernel-devel -y
```

挂载ISO文件
===
```shell
mount -t iso9660 -o loop,user /root/rhel-server-6.4-x86_64-dvd.iso /mnt/cdrom/
```

查看硬盘WWID及HBA卡WWN号
===
## 查看硬盘WWID
```shell
/sbin/scsi_id  -g  -u /dev/sdb
```
## 查看HBA卡WWN
### Qlogic系列的HBA卡
```shell
cat /proc/scsi/qla2xxx/N
```
### 博科的HBA卡
```shell
cd /sys/class/fc_host/hostN
cat port_name
```

yum使用本地CD源
===
```shell
# 首先,挂载光盘
mkdir /mnt/cdrom && mount /dev/cdrom /mnt/cdrom
# 进入yum配置文件目录，备份原来的源文件
cd /etc/yum.respos.d
mkdir yumbak
mv *.repo yumbak
# 在当前目录新建配置文件Centos-iso.repo,
vi centos_iso.repo
  [base]
  name=iso
  baseurl=file:///mnt/cdrom
  gpgcheck=0
    #######

# 刷新yum缓存:
yum clean all
yum makecache
```

chroot之前要做的事情
===
```shell
# LFS是要chroot到达的目录
#在新建的目录中挂载虚拟内核文件系统:
mkdir -pv $LFS/{dev,proc,sys}
#新建初始化设备节点:
mknod -m 600 $LFS/dev/console c 5 1
mknod -m 666 $LFS/dev/null c 1 3
#挂载移植/dev
mount -v --bind /dev $LFS/dev
#挂载虚拟内核文件系统：
mount -vt devpts devpts $LFS/dev/pts
mount -vt tmpfs shm $LFS/dev/shm
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
```

查看进程树的gid
===
```shell
ps x -o  "%p %r %y %x %c "
```

查看进程打开的文件句柄
===
```shell
lsof -n|awk '{print $2}'|sort|uniq -c|sort -nr|more

131 24204　
57 24244　　
57 24231
#其中第一列是打开的句柄数,第二列是进程ID
```
