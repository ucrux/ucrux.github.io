---
author: ucrux
comments: true
date: 2018-04-21 21:31:00 +0000
layout: post
title: docker swarm集群
image: /assets/images/blog_back.jpg
categories:
- 运维
tags:
- docker
---

**组建docker集群**

创建 Swarm 集群
===
## 初始化集群
```shell
# 初始化管理节点
docker swarm init --advertise-addr 172.16.80.41
# 以下为命令输出
To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-3wo3duvgy43l79upyplzoqxs2e4a91xg4rgkj2xenbwe4ht4ir-e0g3rm35dmyi0jkkf0cvmtl8e 172.16.80.41:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

<!-- more -->

## 增加工作节点
```shell
 docker swarm join --token SWMTKN-1-3wo3duvgy43l79upyplzoqxs2e4a91xg4rgkj2xenbwe4ht4ir-e0g3rm35dmyi0jkkf0cvmtl8e 172.16.80.41:2377
```
## 查看集群
```shell
docker node ls
```

## 部署服务
```shell
# 部署
docker service create --replicas 3 -p 80:80 --name nginx nginx:1.13.7-alpine
# 查看 
docker service ls
docker service ps nginx
# 日志
docker service logs nginx
```

## 删除服务
```
docker service rm nginx
```

使用 compose 文件部署服务
===

使用 docker-compose.yml 我们可以一次启动多个关联的服务

## 部署wordpree应用
以下为 docker-compose.yml 文件的内容
```
version: "3"

services:
  wordpress:
    image: wordpress
    ports:
      - 80:80
    networks:
      - overlay
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
    deploy:
      mode: replicated
      replicas: 3

  db:
    image: mysql
    networks:
       - overlay
    volumes:
      - db-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    deploy:
      placement:
        constraints: [node.role == manager]

  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    stop_grace_period: 1m30s
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]

volumes:
  db-data:
networks:
  overlay:
```
### 部署服务
```
docker stack deploy -c docker-compose.yml wordpress
```
### 移除服务
```
docker stack down wordpress
```

### docker-compose.yml 示例
```
version: "3"

services:

  gitlab:
    image: gitlab/gitlab-ce:latest
    hostname: gitlab
    networks:
       - overlay
    ports:
       - 22:22
    volumes:
      - /conf/gitlab/config:/etc/gitlab
      - /conf/gitlab/logs:/var/log/gitlab
      - /conf/gitlab/data:/var/opt/gitlab
    deploy:
      placement:
        constraints: [node.role == manager]

  nginx:
    image: centos:ngx
    hostname: nginx
    ports:
      - 80:80
      - 443:443
    networks:
      - overlay
    volumes:
      - /conf/nginx/ssl:/opt/nginx/ssl
      - /conf/nginx/nginx.conf:/opt/nginx/conf/nginx.conf
      - /conf/nginx/logs:/opt/nginx/logs
      - /conf/nginx/vhosts:/opt/nginx/conf/vhosts
    deploy:
      placement:
        constraints: [node.role == manager]

networks:
  overlay:
```

敏感数据管理
===

## 创建 secret
```shell
openssl rand -base64 20 | docker secret create mysql_passwd -
openssl rand -base64 20 | docker secret create mysql_root_passwd -
```
## 示例
```shell
# 创建网络
docker network create -d overlay mysql_private
# 创建mysql服务
docker service create \
     --name mysql \
     --replicas 1 \
     --network mysql_private \
     --mount type=volume,source=mydata,destination=/var/lib/mysql \
     --secret source=mysql_root_passwd,target=mysql_root_password \
     --secret source=mysql_passwd,target=mysql_password \
     -e MYSQL_ROOT_PASSWORD_FILE="/run/secrets/mysql_root_password" \
     -e MYSQL_PASSWORD_FILE="/run/secrets/mysql_password" \
     -e MYSQL_USER="wordpress" \
     -e MYSQL_DATABASE="wordpress" \
     mysql:latest
# 创建wordpress服务
docker service create \
     --name wordpress \
     --replicas 1 \
     --network mysql_private \
     --publish 30000:80 \
     --mount type=volume,source=wpdata,destination=/var/www/html \
     --secret source=mysql_passwd,target=wp_db_password,mode=0400 \
     -e WORDPRESS_DB_USER="wordpress" \
     -e WORDPRESS_DB_PASSWORD_FILE="/run/secrets/wp_db_password" \
     -e WORDPRESS_DB_HOST="mysql:3306" \
     -e WORDPRESS_DB_NAME="wordpress" \
     wordpress:latest
```

配置文件管理
===

**注意:docker config 仅能在 Swarm 集群中使用**

## 创建config
```shell
# 新建 redis.conf 文件
port 6380
# 创建config
docker config create redis.conf redis.conf
# 查看config
docker config ls
# 创建redis服务
docker service create \
     --name redis \
     # --config source=redis.conf,target=/etc/redis.conf \
     --config redis.conf \
     -p 6379:6380 \
     redis:latest \
     redis-server /redis.conf
```
> 如果你没有在 target 中显式的指定路径时,
> 默认的 redis.conf 以 tmpfs 文件系统挂载到容器的 /config.conf


