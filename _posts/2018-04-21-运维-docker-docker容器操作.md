---
author: ucrux
comments: true
date: 2018-04-21 21:25:00 +0000
layout: post
title: docker容器操作
image: /assets/images/blog_back.jpg
categories:
- 运维
tags:
- docker
---


基本操作
===

## 启动一个镜像
```shell
docker run -it --rm --name ubuntubash ubuntu:16.04 bash
```

<!-- more -->

## 停止一个容器
```shell
docker container stop xxxx
docker container restart xxxx
```
## 启动一个已停止的容器
```shell
docker container start xxxx
```
## 查看容器日志
```shell
docker logs xxxx
```
## 获取在后台(使用 -d 参数)运行的容器
```shell
docker exec -i -t xxxx bash
docker attech xxxx
```
## 容器导入和导出
### 导出
```shell
docker export xxxx > xxxx.tar
```

### 导入
```shell
cat xxxx.tar | docker import - test/xxxx
```

## 删除容器
```shell
docker container rm  xxxx
# 清理所有已停止的容器
docker container prune
```
## 容器运行示例
```shell
# 启动nginx
docker run --detach \
    --hostname nginx \
  --network br_proxy \
    -p 443:443 -p 80:80 \
    --name nginx --restart always \
    --mount type=bind,source=/data/conf/nginx/nginx.conf,target=/opt/nginx/conf/nginx.conf \
  --mount type=bind,source=/data/conf/nginx/vhosts,target=/opt/nginx/conf/vhosts \
  --mount type=bind,source=/data/conf/nginx/logs,target=/opt/nginx/logs \
  --mount type=bind,source=/data/ssl,target=/opt/nginx/ssl \
    cent:ngx
# 启动gitlab
docker run --detach \
    --hostname gitlab \
  --network br_proxy \
    --sysctl net.core.somaxconn=1024 \
    --ulimit sigpending=62793 \
    --ulimit nproc=131072 \
    --ulimit nofile=60000 \
    --ulimit core=0 \
    --publish 22:22 \
    --name gitlab \
    --restart always \
    --mount type=bind,source=/data/conf/gitlab/config,target=/etc/gitlab \
    --mount type=bind,source=/data/ssl,target=/etc/gitlab/ssl \
    --mount type=bind,source=/data/conf/gitlab/logs,target=/var/log/gitlab \
    --mount type=bind,source=/data/conf/gitlab/data,target=/var/opt/gitlab \
    gitlab/gitlab-ce:latest
```



docker-compose使用
===

**定义和运行多个 Docker 容器的应用**

## 安装
```
# 1 下载二进制文件
curl -L https://github.com/docker/compose/releases/download/1.17.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose

# 2 pip安装
pip install -U docker-compose

# 3 使用容器执行
curl -L https://github.com/docker/compose/releases/download/1.8.0/run.sh > /usr/local/bin/docker-compose

# 命令补全
curl -L https://raw.githubusercontent.com/docker/compose/1.8.0/contrib/completion/bash/docker-compose > /etc/bash_completion.d/docker-compose
```

## 卸载
```
rm /usr/local/bin/docker-compose

# or
pip uninstall docker-compose
```

## 使用

### 命令说明
```
docker-compose scale web=3 redis=2  # 指定启动3个web容器, 2个redis容器
```

### docker-compose.yml及标签说明
```
version: "3"

services:
  webapp:
    image: examples/web
    ports:
      - "80:80"
    volumes:
      - "/data"
```

#### build
```
version: '3'
services:

  webapp:
    build:
      context: ./dir
      dockerfile: Dockerfile-alternate
      args:
        buildno: 1
```

> context 指定 Dockerfile 所在文件夹的路径
> dockerfile 指定 Dockerfile 文件
> args 指定镜像构建时的参数

#### cap_add, cap_drop
指定容器的内核能力(capacity)分配
例如:让容器拥有所有能力可以指定为
```
cap_add:
  - ALL
```
去掉 NET_ADMIN 能力可以指定为
```
cap_drop:
  - NET_ADMIN
```

#### devices

指定设备映射关系
```
devices:
  - "/dev/ttyUSB1:/dev/ttyUSB0"
```

#### depends_on
解决容器的依赖,启动先后的问题.以下例子中会先启动 redis db 再启动 web
```
version: '3'

services:
  web:
    build: .
    depends_on:
      - db
      - redis

  redis:
    image: redis

  db:
    image: postgres
```

#### env_file
从文件中读取环境变量
```
env_file: .env

env_file:
  - ./common.env
  - ./apps/web.env
  - /opt/secrets.env
```

#### healthcheck
通过命令检查容器是否健康运行
```
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 1m30s
  timeout: 10s
  retries: 3
```

#### logging
配置日志选项
```
logging:
  driver: syslog
  options:
    syslog-address: "tcp://192.168.0.42:123"
```
目前支持三种日志驱动类型
```
driver: "json-file"
driver: "syslog"
driver: "none"
```
options 配置日志驱动的相关参数
```
options:
  max-size: "200k"
  max-file: "10"
```

#### secrets
存储敏感数据,例如 mysql 服务密码
```
version: "3"
services:

mysql:
  image: mysql
  environment:
    MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
  secrets:
    - db_root_password
    - my_other_secret

secrets:
  my_secret:
    file: ./my_secret.txt
  my_other_secret:
    external: true
```

#### sysctls
配置容器内核参数
```
sysctls:
  net.core.somaxconn: 1024
  net.ipv4.tcp_syncookies: 0

sysctls:
  - net.core.somaxconn=1024
  - net.ipv4.tcp_syncookies=0
```

#### ulimits
指定容器的 ulimits 限制值
例如,指定最大进程数为 65535,
指定文件句柄数为 20000(软限制,应用可以随时修改,不能超过硬限制) 
和 40000(系统硬限制,只能 root 用户提高)
```
  ulimits:
    nproc: 65535
    nofile:
      soft: 20000
      hard: 40000
```

#### volumes
数据卷所挂载路径设置.
可以设置宿主机路径 (HOST:CONTAINER) 
或加上访问模式 (HOST:CONTAINER:ro).
该指令中路径支持相对路径
```
volumes:
 - /var/lib/mysql
 - cache/:/tmp/cache
 - ~/configs:/etc/configs/:ro
```
