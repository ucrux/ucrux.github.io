---
author: asmbits
comments: true
date: 2018-04-21 21:20:00 +0000
layout: post
title: nginx安装brotil压缩模块
image: /assets/images/blog_back.jpg
categories:
- 运维
tags:
- nginx
---

**Brotli module requires recent version of NGINX (1.9.11+).**

<!-- more -->

## 安装依赖
```
yum install -y libtool
git clone https://github.com/bagder/libbrotli
cd libbrotli
./autogen.sh
./configure --prefix=/usr
make  && make install
ldconfig

wget http://luajit.org/download/LuaJIT-2.0.3.tar.gz
tar xf LuaJIT-2.0.3.tar.gz
cd LuaJIT-2.0.3
make
make install
ln -s /usr/local/lib/libluajit-5.1.so.2 /lib64/libluajit-5.1.so.2
```

## 安装模块
```
cd /usr/local/src
git clone https://github.com/google/ngx_brotli
cd ngx_brotli
git submodule update --init

wget http://soft.ik8s.com/nginx/nginx-1.10.3.tar.gz

# 查看以前的编译参数
/usr/local/nginx/sbin/nginx -V

CFLAGS="I/usr/local/sharelib/brotli/include \
./configure \
--prefix=/usr/local/nginx \
--with-openssl=/usr/local/src/openssl-1.0.1s \
--with-http_ssl_module \
--with-pcre --with-http_gzip_static_module \
--add-module=/usr/local/src/echo-nginx-module-0.59 \
--add-module=/usr/local/src/lua-nginx-module-master \
--add-module=/usr/local/src/ngx_devel_kit-master \
--add-module=/usr/local/src/redis2-nginx-module-master \
--add-module=/usr/local/src/set-misc-nginx-module-master \
--add-module=/usr/local/src/nginx-push-stream-module-master --with-http_realip_module --with-http_sub_module \
--add-module=/usr/local/src/ngx_brotli

make -j4 

cp obj

# 替换原来 nginx 执行文件
# 平滑升级
kill -USR2 {原来nginx master进程的pid}
kill -WINCH {原来nginx master进程的pid}
# 如果升级不行
kill -HUP {原来nginx master进程的pid}
kill -TERM {新nginx master进程的pid}
# 如果升级成功
kill -QUIT {原来nginx master进程的pid}
```
