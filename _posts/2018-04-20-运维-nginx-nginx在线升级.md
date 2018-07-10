---
author: ucrux
comments: true
date: 2018-04-20 17:18:32 +0000
layout: post
title: nginx在线升级
image: /assets/images/blog_back.jpg
categories:
- 运维
tags:
- nginx
---

## 第一步,替换二进制文件,启动新的进程

- 用新的nginx可执行文件替换旧的可执行文件
- USR2 信号发送给正在运行的nginx的主进程
  - kill -USR2 $pid
- 主进程首先用 .oldbin 为后缀,重命名 pidfile 
  - 例如 /usr/local/nginx/logs/nginx.pid.oldbin
- 新的可执行文件会开启新的主进程和worker进程

<!-- more -->

```shell
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1164 pause  nginx: master process /usr/local/nginx/sbin/nginx
33134 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
33135 33126 nobody   0.0  1380 kqread nginx: worker process (nginx)
33136 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
36264 33126 root     0.0  1148 pause  nginx: master process /usr/local/nginx/sbin/nginx
36265 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36266 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36267 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
```

## 第二步,平滑退出旧的主进程

- 所有的worker进程 (old and new ones) 都接受请求 
- 如果 kill -WINCH $pid_old_master
  - 旧的主进程就会让他的worker进程停止接受新请求,并让其平滑退出

```shell
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1164 pause  nginx: master process /usr/local/nginx/sbin/nginx
36264 33126 root     0.0  1148 pause  nginx: master process /usr/local/nginx/sbin/nginx
36265 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36266 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36267 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
```

## 第三部,确认升级

- 旧的主进程并不会停止他的sockets,如果需要它还可以重新启动他的工作进程
- 如果新的主进程及其worker进程没有按预期工作,可以执行如下几步:
  - kill -HUP $pid_old_master
    - 旧的主进程会重新读取配置文件和启动worker进程
    - kill -QUIT $pid_new_master,新的主进程下worker进程将平滑退出
  - kill -TERM $pid_new_master 
    - 新的主进程及其worker进程会立即退出 
    - 旧的主进程将自动启动woker进程
    - 新的主进程退出后,旧的主进程将会把pidfile重新命名之前的名字
- 如果升级成功
  - kill -QUIT $pid_old_master
  - 只有新的主进程及其worker进程会被保留

```shell
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
36264     1 root     0.0  1148 pause  nginx: master process /usr/local/nginx/sbin/nginx
36265 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36266 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36267 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
```
