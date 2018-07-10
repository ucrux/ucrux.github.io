---
author: asmbits
comments: true
date: 2018-04-21 21:29:00 +0000
layout: post
title: docker镜像管理
image: /assets/images/blog_back.jpg
categories:
- 运维
tags:
- docker
---

镜像基础管理
===

## 获取镜像

- docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]

```shell
# 获取ubuntu 16.04
docker pull ubuntu:16.04
```

> 由于没有指定地址及仓库名<br>
> 所以使用默认地址,及默认仓库library/ubuntu<br>

<!-- more -->

```shell
## 测试运行
docker run -it --rm ubuntu:16.04 bash
```

> -i 交互<br>
> -t 终端<br>
> --rm 退出容器后随即删除<br>

## 列出镜像
```shell
docker image ls
# 查看 中间层镜像
docker image ls -a
# 列出部分镜像
docker image ls ubuntu
docker image ls ubuntu:16.04
# 查看在 mongo:3.2 之后建立的镜像
docker image ls -f since=mongo:3.2
```

## 镜像体积
```shell
docker system df
```

## 虚悬镜像

> 仓库名 和 标签 均为 <none>

```shell
# 查看此类镜像
docker image ls -f dangling=true
# 清除此类镜像
docker image prune
```

## 利用commit理解镜像构成
```shell
# 运行一个nginx容器
docker run --name webserver -d -p 80:80 nginx
```
> --name 将容器命名为 webserver 
> -d daemon
> -p 80:80 端口映射
> nginx 为镜像名

### 进入 webserver 容器,修改首页
```shell
docker exec -it webserver bash
echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
exit
# 查看容器存储层的更改
docker diff webserver
```

### 保存容器为新的镜像

- docker commit [选项] <容器ID或容器名> [<仓库名>[:<标签>]]

```shell
# 将容器名为 webserver 的容器保存为镜像 nginx:v2
docker commit \
    --author "chad" \
    --message "change index.html" \
    webserver \
    nginx:v2
# 查看镜像历史记录
docker history nginx:v2
```

### 运行此新镜像
```shell
docker run --name web2 -d -p 81:80 nginx:v2
```

## 传统镜像迁移方法

> 无法保存 EXPOSE, CMD, ENTRYPOINT 信息

```shell
# 保存镜像的命令为
docker save alpine | gzip > alpine-latest.tar.gz
# 然后我们将 alpine-latest.tar.gz 文件复制到了到了另一个机器上
# 可以用下面这个命令加载镜像
docker load -i alpine-latest.tar.gz
Loaded image: alpine:latest
```


使用Dockerfile定制镜像
===

> Dockerfile 是一个文本文件<br>
> 其内包含了一条条的指令(Instruction)<br>
> 每一条指令构建一层<br>
> 因此每一条指令的内容,就是描述该层应当如何构建<br>

## Dockerfile命令

### COPY 复制文件

- COPY <源路径>... <目标路径>
- COPY ["<源路径1>",... "<目标路径>"]

> <目标路径> 可以是容器内的绝对路径<br>
> 也可以是相对于工作目录的相对路径<br>
> 工作目录可以用 WORKDIR 指令来指定<br>
> WORKDIR将影响其后的 CMD, ENTRYPOINT, RUN, ADD, COPY等指令

```shell
COPY package.json /usr/src/app/

# 原路径可以是多个,可以是通配符
COPY hom* /mydir/
COPY hom?.txt /mydir/
```

### ADD 更高级的复制文件

> 会自动解压的 COPY, 不推荐使用

```shell
FROM scratch
ADD ubuntu-xenial-core-cloudimg-amd64-root.tar.gz /
```

### CMD 容器启动命令
- shell 格式:CMD <命令>
- exec 格式:CMD ["可执行文件", "参数1", "参数2"...]
- 参数列表格式:CMD ["参数1", "参数2"...]. 
  - 在指定了 ENTRYPOINT 指令后,用 CMD 指定具体的参数


### ENTRYPOINT 入口点

当指定了 ENTRYPOINT 后, CMD 的含义就发生了改变. 不再是直接的运行其命令, 而是将 CMD 的内容作为参数传给 ENTRYPOINT 指令. 换句话说实际执行时,将变为:
```
<ENTRYPOINT> "<CMD>"
```

### ENV 设置环境变量

- ENV key value
- ENV key1=value1 key2=value2 ...

```shell
ENV VERSION=1.0 DEBUG=on \
    NAME="Happy Feet"
```

### ARG 构建参数

- ARG <参数名>[=<默认值>]

构建参数和 ENV 的效果一样, 都是设置环境变量. 所不同的是, ARG所设置的构建环境的环境变量, 在将来容器运行时是不会存在这些环境变量的.

### VOLUME 定义匿名卷

- VOLUME ["<路径1>", "<路径2>"...]
- VOLUME <路径>

```shell
VOLUME /data
# 讲命名卷挂载到 /data 上
docker run -d -v mydata:/data xxxx
```

### EXPOSE 声明端口

- 格式为 EXPOSE <端口1> [<端口2>...]。

### WORKDIR 指定工作目录

- WORKDIR <工作目录路径>

### USER 指定当前用户

- USER <用户名>


### HEALTHCHECK 健康检查

- HEALTHCHECK [选项] CMD <命令>: 设置检查容器健康状况的命令
- HEALTHCHECK NONE: 如果基础镜像有健康检查指令, 使用这行可以屏蔽掉其健康检查指令

```shell
FROM nginx
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
# 5s一次使用 curl 对 nginx 进行健康检查
HEALTHCHECK --interval=5s --timeout=3s \
  CMD curl -fs http://localhost/ || exit 1
```
#### 查看将康检查日志
```shell
docker inspect --format '{{json .State.Health}}' web | python -m json.tool
```

### ONBIULD 

构建基础镜像时不会执行, 构建下级镜像时才会执行, 即本次构建时不会执行, 在以本次构建的镜像为基础的再次构建, 才会执行 ONBUILD 中的指令

```shell
FROM node:slim
RUN mkdir /app
WORKDIR /app
ONBUILD COPY ./package.json /app
ONBUILD RUN [ "npm", "install" ]
ONBUILD COPY . /app/
CMD [ "npm", "start" ]
```


## 以centos为基础构建nginx镜像

```shell
mkdir nginx
cd nginx
touch Dockerfile
vi Dockerfile
```

### 在Dockerfile中添加如下内容

```
FROM centos:latest

COPY software /software

RUN yum install -y gcc gcc-c++ make autoconf git autoconf automake libtool && \
    groupadd -g 500 web && \
    useradd -g web -u 500 -M -s /sbin/nologin web && \
    cd /software && \
    cd libbrotli && \
    ./autogen.sh && \
    ./configure --prefix=/usr && \
    make && \
    make install && \
    ldconfig && \
    cd .. && \
    cd LuaJIT-2.0.3 && \
    make && \
    make install && \
    ln -s /usr/local/lib/libluajit-5.1.so.2 /lib64/libluajit-5.1.so.2 && \
    cd ../nginx-1.12.2 && \
    ./configure \
    --prefix=/opt/nginx \
    --user=web --group=web \
    --with-openssl=/software/openssl-1.0.2n \
    --with-http_ssl_module \
    --with-pcre=/software/pcre-8.38 \
    --with-http_gzip_static_module \
    --with-zlib=/software/zlib-1.2.11 \
    --add-module=/software/echo-nginx-module-0.61 \
    --add-module=/software/redis2-nginx-module-0.14 \
    --add-module=/software/nginx-push-stream-module-0.5.4 \
    --with-http_realip_module --with-http_sub_module \
    --add-module=/software/ngx_brotli \
    --add-module=/software/ngx_http_lower_upper_case \
    --with-http_stub_status_module && \
    make && \
    make install && \
    cd / && \
    rm -rf /software && \
    mkdir -pv /data/nginx/logs && \
    cp -R /opt/nginx/conf /data/nginx/ && \
    cp -R /opt/nginx/html/ /data/nginx/ && \
    yum remove -y autoconf automake gcc gcc-c++ git libtool make \
        cpp fipscheck fipscheck-lib glibc-devel glibc-headers \
        groff-base kernel-headers less libedit libgnome-keyring libgomp libmpc \
        libstdc++-devel m4 mpfr openssh openssh-clients perl perl-Carp perl-Data-Dumper \
        perl-Encode perl-Error perl-Exporter perl-File-Path perl-File-Temp perl-Filter \
        perl-Getopt-Long perl-Git perl-HTTP-Tiny perl-PathTools perl-Pod-Escapes perl-Pod-Perldoc \
        perl-Pod-Simple perl-Pod-Usage perl-Scalar-List-Utils perl-Socket perl-Storable perl-TermReadKey \
        perl-Test-Harness perl-Text-ParseWords perl-Thread-Queue perl-Time-HiRes perl-Time-Local \
        perl-constant perl-libs perl-macros perl-parent perl-podlators perl-threads \
        perl-threads-shared rsync && \
    yum clean all


EXPOSE 80
EXPOSE 443

STOPSIGNAL SIGTERM

CMD ["/opt/nginx/sbin/nginx", "-g", "daemon off;"]
```

> 很多人初学 Docker 制作出了很臃肿的镜像的原因之一,
> 就是忘记了每一层构建的最后一定要清理掉无关文件

### 构建镜像
```shell
docker build -t centos:ngx .
```

#### 如果需要在构建镜像时压缩镜像层数
##### 修改docker配置文件

> 如果没有此文件,请创建

```shell
vi /etc/docker/daemon.json
# 添加如下内容
{
    "experimental": true
}
```

##### 重启docker
```shell
systemctl restart docker
```

##### 构建镜像
```shell
docker build --squash -t centos:ngx .
```

##### 清理多余镜像层
```shell
docker image prune
```

docker仓库使用
===

```shell
# 查找镜像,显示收藏数在N以上的
docker search --filter=stars=N centos 
# 获取镜像
docker pull centos
# 推送镜像
docker tag ubuntu:17.10 username/ubuntu:17.10
```


## 私有仓库

### 安装运行 docker-registry
```shell
docker run -d \
    -p 5000:5000 \
    -v /opt/data/registry:/var/lib/registry \
    registry
```
> -v /opt/data/registry:/var/lib/registry 将镜像文件保存在本地的/opt/data/registry 目录下

### 将镜像推送/获取到本地仓库
```shell
# 标记需上传的镜像
docker tag nginx:v3 127.0.0.1:5000/nginx:v3

# 上传镜像到本地仓库
docker push 127.0.0.1:5000/nginx:v3

# 查看仓库中的镜像
curl 127.0.0.1:5000/v2/_catalog

# 获取本地仓库镜像
docker pull 127.0.0.1:5000/nginx:v3
```

### 取消仓库必须 https 方式访问
```shell
vi /etc/docker/daemon.json 
#中写入如下内容(如果文件不存在请新建该文件)

{
  "registry-mirror": [
    "https://registry.docker-cn.com"
  ],
  "insecure-registries": [
    "192.168.199.100:5000"
  ]
}

# 192.168.199.100:5000 这是需要取消https访问的地址
```

### 使用自签发证书搭建 docker 仓库

```shell
# 创建CA私钥
openssl genrsa -out "root-ca.key" 4096
# 利用私钥创建 CA 根证书请求文件。
openssl req \
        -new -key "root-ca.key" \
        -out "root-ca.csr" -sha256 \
        -subj '/C=CN/ST=Shanxi/L=Datong/O=Your Company Name/CN=Your Company Name Docker Registry CA'

```

> 以上命令中 -subj 参数里的 /C 表示国家,如 CN;
> /ST 表示省;
> /L 表示城市或者地区;
> /O 表示组织名
> /CN 通用名称

```shell
# 配置 CA 根证书,新建 root-ca.cnf
[root_ca]
basicConstraints = critical,CA:TRUE,pathlen:1
keyUsage = critical, nonRepudiation, cRLSign, keyCertSign
subjectKeyIdentifier=hash
# 签发根证书
openssl x509 -req  -days 3650  -in "root-ca.csr" \
        -signkey "root-ca.key" -sha256 -out "root-ca.crt" \
        -extfile "root-ca.cnf" -extensions \
        root_ca
# 生成站点 SSL 私钥
openssl genrsa -out "docker.domain.com.key" 4096
# 使用私钥生成证书请求文件。
openssl req -new -key "docker.domain.com.key" -out "site.csr" -sha256 \
        -subj '/C=CN/ST=Shanxi/L=Datong/O=Your Company Name/CN=docker.domain.com'
# 配置证书,新建 site.cnf 文件
[server]
authorityKeyIdentifier=keyid,issuer
basicConstraints = critical,CA:FALSE
extendedKeyUsage=serverAuth
keyUsage = critical, digitalSignature, keyEncipherment
subjectAltName = DNS:docker.domain.com, IP:127.0.0.1
subjectKeyIdentifier=hash
# 签署站点 SSL 证书。
openssl x509 -req -days 750 -in "site.csr" -sha256 \
        -CA "root-ca.crt" -CAkey "root-ca.key"  -CAcreateserial \
        -out "docker.domain.com.crt" -extfile "site.cnf" -extensions server
```

这样已经拥有了 docker.domain.com 的网站 SSL 私钥 docker.domain.com.key 和 SSL 证书 docker.domain.com.crt<br>
新建 ssl 文件夹并将 docker.domain.com.key docker.domain.com.crt 这两个文件移入，删除其他文件.<br>

#### 修改私有仓库配置文件
```
vi config.yml
# 之后将其挂载到仓库容器中(/etc/docker/registry/config.yml)
version: 0.1
log:
  accesslog:
    disabled: true
  level: debug
  formatter: text
  fields:
    service: registry
    environment: staging
storage:
  delete:
    enabled: true
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
auth:
  htpasswd:
    realm: basic-realm
    path: /etc/docker/registry/auth/nginx.htpasswd
http:
  addr: :443
  host: https://docker.domain.com
  headers:
    X-Content-Type-Options: [nosniff]
  http2:
    disabled: false
  tls:
    certificate: /etc/docker/registry/ssl/docker.domain.com.crt
    key: /etc/docker/registry/ssl/docker.domain.com.key
health:
  storagedriver:
    enabled: true
    interval: 10s
threshold: 3
```

#### 生成http认证文件
```
mkdir auth
docker run --rm \
    --entrypoint htpasswd \
    registry \
    -Bbn username password > auth/nginx.htpasswd
```
#### 编辑 docker-compose.yml
```
version: '2'

services:
  registry:
    image: registry
    ports:
      - "443:443"
    volumes:
      - ./:/etc/docker/registry
      - registry-data:/var/lib/registry

volumes:
  registry-data:
```

#### 启动
```
docker-compose up -d
```
