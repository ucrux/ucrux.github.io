---
author: asmbits
comments: true
date: 2018-04-21 21:23:32 +0000
layout: post
title: docker快速入门
image: /assets/images/blog_back.jpg
categories:
- 运维
tags:
- docker
---

**本例中的使用基于debian9**

docker安装
===
## 卸载旧版的docker
```shell
apt-get remove docker \
        docker-engine \
        docker.io
```

<!-- more -->

## 安装
### 安装依赖
```shell
apt-get update

apt-get install \
        apt-transport-https \
        ca-certificates \
        curl \
        gnupg2 \
        lsb-release \
        software-properties-common
```

### 添加软件源
```shell
# 添加gpg key
## 国内
$ curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/debian/gpg | apt-key add -
## 官方
curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -

# 添加软件源
## 国内源
add-apt-repository \
   "deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/$(. /etc/os-release; echo "$ID") \
   $(lsb_release -cs) \
   stable"
## 官方源
add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/debian \
  $(lsb_release -cs) \
  stable"  
```

> 以上命令会添加稳定版本的 Docker CE APT 镜像源
> 如果需要最新或者测试版本的 Docker CE 请将 stable 改为 edge 或者 test
> 从 Docker 17.06 开始,edge test 版本的 APT 镜像源也会包含稳定版本的 Docker CE

### 安装
```shell
apt-get update
apt-get install docker-ce
```


简单配置
===

### 启动docker
```shell
systemctl enable docker
systemctl start docker
```

### 添加docker用户和组

> 默认情况下,docker 命令会使用 Unix socket 与 Docker 引擎通讯
> 而只有 root 用户和 docker 组的用户才可以访问 Docker 引擎的 Unix socket
> 出于安全考虑,一般 Linux 系统上不会直接使用 root 用户组
> 因此,更好地做法是将需要使用 docker 的用户加入 docker 用户组

```shell
#groupadd -g 500 docker # debian9安装docker时,用户组自动建好了
useradd -m -s /bin/bash -u 500 dock
usermod -aG docker dock
```

### 测试docker

> 使用dock用户组中的用户登陆

```shell
su - dock
docker run hello-world
```

镜像简单管理
===
```shell
# 搜索centos镜像
docker search centos
# 获取centos镜像
docker pull centos
```

docker-machine使用
===

## 安装
```shell
curl -L https://github.com/docker/machine/releases/download/v0.13.0/docker-machine-`uname -s`-`uname -m` > /usr/local/bin/docker-machine

chmod +x /usr/local/bin/docker-machin
```

## 使用
```shell
# 使用virtualbox作为后台驱动
docker-machine create -d virtualbox test
# 使用xhyve (macos)
brew install docker-machine-driver-xhyve
docker-machine create \
      -d xhyve \
      # --xhyve-boot2docker-url ~/.docker/machine/cache/boot2docker.iso \
      --engine-opt dns=114.114.114.114 \
      --engine-registry-mirror https://registry.docker-cn.com \
      --xhyve-memory-size 2048 \
      --xhyve-rawdisk \
      --xhyve-cpu-count 2 \
      xhyve
# 注意:
# 非首次创建时建议加上 
# --xhyve-boot2docker-url ~/.docker/machine/cache/boot2docker.iso 参数,
# 避免每次创建时都从 GitHub 下载 ISO 镜像
# 更多参数请使用 docker-machine create --driver xhyve --help 命令查看
```
