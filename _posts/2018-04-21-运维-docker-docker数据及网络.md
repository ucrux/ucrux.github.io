---
author: asmbits
comments: true
date: 2018-04-21 21:27:00 +0000
layout: post
title: docker数据及网络
image: /assets/images/blog_back.jpg
categories:
- 运维
tags:
- docker
---


数据管理
===

## 数据卷
```shell
docker volume create my-vol
# 查看卷
docker volume ls
# 查看my-vol的详细信息
docker valume inspect my-vol
# 创建一个名为 web 的容器, 并加载一个 数据卷 到容器的 /webapp 目录
docker run -d -P \
    --name web \
    # -v my-vol:/wepapp \
    --mount source=my-vol,target=/webapp \
    training/webapp \
    python app.py
# 查看容器信息
docker inspect web
# 删除卷
docker volume rm my-vol
# 清理无主卷
docker volume prune
```

<!-- more -->

## 挂载本地目录
```shell
docker run -d -P \
    --name web \
    # -v /src/webapp:/opt/webapp \
    --mount type=bind,source=/src/webapp,target=/opt/webapp \
    training/webapp \
    python app.py
# 只读方式挂载
docker run -d -P \
    --name web \
    # -v /src/webapp:/opt/webapp:ro \
    --mount type=bind,source=/src/webapp,target=/opt/webapp,readonly \
    training/webapp \
    python app.py
# 挂载单个文件
docker run --rm -it \
   # -v $HOME/.bash_history:/root/.bash_history \
   --mount type=bind,source=$HOME/.bash_history,target=/root/.bash_history \
   ubuntu:17.10 \
   bash
```

网络管理
===

- 使用 **-P** 参数宿主机的 *高端口* 会随机映射到容器开放的端口 
- -p 则可以指定要映射的端口.并且,在一个指定端口上只可以绑定一个容器
  - ip:hostPort:containerPort | ip::containerPort | hostPort:containerPort
  - docker run -d -p 5000:5000 training/webapp python app.py
  - docker run -d -p 127.0.0.1:5000:5000 training/webapp python app.py
  - docker run -d -p 127.0.0.1::5000 training/webapp python app.py
    - 绑定宿主机127.0.0.1任意端口到容器的5000端口
  - docker run -d -p 127.0.0.1:5000:5000/udp training/webapp python app.py
    - 绑定udp端口
- docker port nostalgic_morse 5000
  - 查看绑定的端口 

```shell
# 绑定多个端口
docker run -d \
    -p 5000:5000 \
    -p 3000:80 \
    training/webapp \
    python app.py
```

## 自定义docker网络

### 新建网络
```shell
# -d 指定网络类型, [bridge|overlay]
docker network create -d bridge my-net
```
### 运行容器,连接网络
```
docker run -it --rm --name busybox1 --network my-net busybox sh
docker run -it --rm --name busybox2 --network my-net busybox sh
```
## 配置DNS
配置全部容器的 DNS,可以在 /etc/docker/daemon.json 文件中增加以下内容来设置
```
{
  "dns" : [
    "114.114.114.114",
    "8.8.8.8"
  ]
}
```

docker run
  - -h HOSTNAME | --hostname=HOSTNAME 会被写到容器的/etc/hosts /etc/hostname
  - --dns=IP_ADDR 会被写到容器的 /etc/resolv.conf
  - --dns-search=DOMAIN 设定容器的搜索域,当设定搜索域为 .example.com 时,在搜索一个名为 host 的主机时,DNS 不仅搜索 host,还会搜索 host.example.com


## 高级网络配置

### 自定义网桥
#### 删除默认网桥
```shell
systemctl stop docker
ip link set dev docker0 down
brctl delbr docker0       # apt-get install bridge-utils
```
#### 创建一个新的网桥
```shell
brctl addbr br01
ip addr add 10.0.0.0/24 dev br01 
ip link set dev br01 up
```
#### 修改 docker 配置文件 /etc/docker/daemon.json , 添加
```
{
  "bridge": "br01"
}
```
#### 启动服务
```shell
systemctl start docker
```

### 创建点对点连接
#### 启动2个容器
```shell
docker run --rm -it --net=none bash /bin/bash
docker run --rm -it --net=none bash /bin/bash
```
#### 找到进程号,然后创建网络命名空间的跟踪文件
```shell
docker inspect -f '{{.State.Pid}}' d49a4864de03  #容器ID
# 11515
docker inspect -f '{{.State.Pid}}' 0186585210c3
# 11433

mkdir -p /var/run/netns
ln -s /proc/11515/ns/net /var/run/netns/11515
ln -s /proc/11433/ns/net /var/run/netns/11433
```
#### 创建一对 peer 接口,然后配置路由
```shell
ip link add A type veth peer name B

ip link set A netns 11515
ip netns exec 11515 ip addr add 10.1.1.1/32 dev A
ip netns exec 11515 ip link set A up
ip netns exec 11515 ip route add 10.1.1.2/32 dev A

ip link set B netns 11433
ip netns exec 11433 ip addr add 10.1.1.2/32 dev B
ip netns exec 11433 ip link set B up
ip netns exec 11433 ip route add 10.1.1.1/32 dev B
```

## 将为配置网络的容器,加入现有桥接网卡
```shell
# 创建容器
docker run -it --rm base --net=none  /bin/bash
# 查找容器的pid,并建立网络命名空间
docker inspect -f '{{.State.Pid}}' $containerID
mkdir -pv /var/run/netns
ln -sv /proc/$pid/ns/net /var/run/netns/$pid
# 检查桥接网络的ip和子网掩码
ip add show docker0
# 创建一对 “veth pair” 接口 A 和 B,绑定 A 到网桥 docker0,并启用它
ip link add A type veth peer name B
brctl addif docker0 A
ip link set A up
# 将B放到容器的网络命名空间,命名为 eth0,启动它并配置一个可用 IP(桥接网段)和默认网关
ip link set B netns $pid
ip netns exec $pid ip link set B name eth0
ip netns exec $pid ip link set eth0 up
ip netns exec $pid ip addr add 172.17.0.3/24 dev eth0 
ip netns exec $pid ip route add default via 172.17.0.1
```
