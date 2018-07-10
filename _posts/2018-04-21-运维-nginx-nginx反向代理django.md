---
author: asmbits
comments: true
date: 2018-04-21 21:18:32 +0000
layout: post
title: nginx反向代理django
image: /assets/images/blog_back.jpg
categories:
- 运维
tags:
- nginx
---

## 安装uwsji
```shell
./pip2.7 install uwsgi
```

<!-- more -->

## 配置uwsji启动配置文件
```shell
# Django-related settings

socket = :8001

# the base directory (full path)
chdir           = /home/web/blog

# Django s wsgi file
module          = blog.wsgi

# process-related settings
# master
master          = true

# maximum number of worker processes
processes       = 4

# ... with appropriate permissions - may be needed
# chmod-socket    = 664
# clear environment on exit
vacuum          = true

# 详情请 uwsji --help
```

## nginx配置文件
```
server {
    # the port your site will be served on
    listen      80;
    # the domain name it will serve for
    server_name 127.0.0.1; # substitute your machine's IP address or FQDN
    charset     utf-8;

    # max upload size
    client_max_body_size 75M;   # adjust to taste

    # Django media
    location /media  {
        alias /home/web/blog/media;  # your Django project's media files - amend as required
    }

    location /static {
        alias /home/web/blog/static; # your Django project's static files - amend as required
    }

    # Finally, send all non-media requests to the Django server.
    location / {
        include     uwsgi_params; # the uwsgi_params file you installed
        uwsgi_pass 127.0.0.1:8001;
    }
}
```
