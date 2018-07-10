---
author: asmbits
comments: true
date: 2018-04-21 21:20:00 +0000
layout: post
title: nginx快速入门
image: /assets/images/blog_back.jpg
categories:
- 运维
tags:
- nginx
---

documentation
===
<a href=http://nginx.org/en/docs/>官方文档</a>

<!-- more -->

install
===

## get packages
```
# ngx_http_gzip_module needed
wget http://zlib.net/zlib-1.2.11.tar.gz
# ngx_http_rewrite_module needed
wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.40.tar.gz

wget http://nginx.org/download/nginx-1.12.1.tar.gz
```

## install packages
```
tar xf pcre-8.40.tar.gz && \
tar xf zlib-1.2.11.tar.gz && \
tar xf nginx-1.12.1.tar.gz 

cd nginx-1.12.1
#编译选项 http://nginx.org/en/docs/configure.html
./configure --prefix=/opt/nginx-1.12.1 \
--with-http_ssl_module \
--with-pcre=../pcre-8.40 \
--with-pcre-jit \
--with-zlib=../zlib-1.2.11

make -j4 && make install

ln -sv /opt/nginx-1.12.1 /opt/nginx
```

## config file
```
pcre_jit on ;
user web web ;

worker_processes 4 ;
worker_cpu_affinity 0001 0010 0100 1000 ;
worker_priority -10 ;
worker_rlimit_nofile 2048 ;     #about worker_connections

events {
        use epoll ;
        worker_aio_requests 128 ;
        multi_accept on ;
        worker_connections 1024 ;
}

include vhosts/*.conf ;

http {
        server {
                listen 8080 ;
                root /data/up1 ;

                location / {

                }
        }

        server {
                location / {
                        proxy_pass http://10.0.0.100:8080 ;
                }
                location ~ \.(gif|jpg|png)$ {
                        root /data/images ;
                }
        }
}
```

startup
===

```
sbin/nginx

ps -ef | grep nginx 
root      14629      1  0 17:41 ?        00:00:00 nginx: master process ./nginx
nobody    14630  14629  0 17:41 ?        00:00:00 nginx: worker process
```

**The main purpose of the master process is to read and evaluate configuration, and maintain worker processes. Worker processes do actual processing of requests**

>about config file structure
>The events and http directives reside in the main context, server in http, and location in server.


```
nginx -s signal
```

- stop : fast shutdown (or kill -s [TERM,INT] <master pid>)
- quit : graceful shutdown  (or kill -s QUIT <master pid>)
- reload : reloading config file (or kill -s HUP <master pid>)
- reopen : reopen log files (or kill -s USR1 <master pid>)

serving static content
===

- /data/www (HTML files) 
- /data/images (images)

```
mkdir -pv /data/{www,images}
```

- config file
  - http
    - server
    - location (two)

```
vi conf/nginx.conf
# must have events and http

events {
    worker_connections  1024;
}

http {
        server {
                location / {
                        root /data/www ;
                }
                location /images/ {
                        root /data ;
                }
        }
}

```

Setting Up a Simple Proxy Server
===

- config file

```
events {
    worker_connections  1024;
}

http {
        server {
                listen 8080 ;
                root /data/up1 ;
                        
                location / {
                                
                }       
        }       

        server {
                location / {
                        # 会访问到/data/up1中
                        proxy_pass http://10.0.0.100:8080 ;
                }       
                #  A regular expression should be preceded with ~
                location ~ \.(gif|jpg|png)$ {       #匹配后缀名
                        root /data/images ; 
                }       
        }       
}
```

Setting Up FastCGI Proxying
===

*nginx can be used to route requests to FastCGI servers which run applications built with various frameworks and programming languages such as PHP*

- The most basic nginx configuration to work with a FastCGI server includes using the *fastcgi_pass* directive instead of the *proxy_pass* directive.
- *fastcgi_param* directives to set parameters passed to a FastCGI server. 
-  In PHP
  -  *SCRIPT_FILENAME* parameter is used for determining the script name 
  -  *QUERY_STRING* parameter is used to pass request parameters


```
server {
    location / {
        fastcgi_pass  localhost:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param QUERY_STRING    $query_string ;
    }

    location ~ \.(gif|jpg|png)$ {
        root /data/images ;
    }
}
```


>This will set up a server that will route all requests except for requests for static images to the proxied server operating on localhost:9000 through the FastCGI protocol.<br>


跨域访问
===

```
## location
if ($request_method = 'OPTIONS') {
                add_header 'Access-Control-Allow-Origin' '*';
                add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
                
                add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
                
                add_header 'Access-Control-Max-Age' 1728000;
                add_header 'Content-Type' 'text/plain charset=UTF-8';
                add_header 'Content-Length' 0;
                return 204;
             }
             if ($request_method = 'POST') {
                add_header 'Access-Control-Allow-Origin' '*';
                add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
                add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
             }
             if ($request_method = 'GET') {
                add_header 'Access-Control-Allow-Origin' '*';
                add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
                add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
             }
```
