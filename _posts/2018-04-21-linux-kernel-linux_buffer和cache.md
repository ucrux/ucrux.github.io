---
author: asmbits
comments: true
date: 2018-04-21 14:28:00 +0000
layout: post
title: linux中的buffer和cache
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- kernel
---

什么是buffer/cache
===
```shell
free -m
             total       used       free     shared    buffers     cached
Mem:          3832        106       3725          0          4         25
-/+ buffers/cache:         76       3755
Swap:          511          0        511

```

1. buffer 指 Linux 内存的： Buffer cache (缓冲区缓存)
2. cache 指 Linux 内存中的： Page cache  (页面缓存)

<!-- more -->

**如果有内存是以 page 进行分配管理的,使用 page cache 作为其缓存来管理使用**
**如果有内存是以 block 进行分配管理的,使用 Buffer cache 作为其缓存来管理使用**

## 什么是 page cache
Page cache 主要用来作为文件系统上的文件数据的缓存来用,尤其是针对当进程对文件有 read ／ write 操作的时候
在当前的系统实现里,page cache 也被作为其它文件类型的缓存设备来用,所以事实上 page cache 也负责了大部分的块设备文件的缓存工作

## 什么是 buffer cache
Buffer cache 则主要是设计用来在系统对块设备进行读写的时候，对块进行数据缓存的系统来使用

**一般情况下两个缓存系统是一起配合使用的,比如当我们对一个文件进行写操作的时候, page cache 的内容会被改变,而 buffer cache 则可以用来将 page 标记为不同的缓冲区,并记录是哪一个缓冲区被修改了.这样,内核在后续执行脏数据的回写(writeback)时,就不用将整个 page 写回,而只需要写回修改的部分即可**

如何回收 cache
==
## 自动触发
Linux 内核会在内存将要耗尽的时候,触发内存回收的工作

## 手动触发
对 /proc/sys/vm/drop_caches 文件进行操作
```shell
cat /proc/sys/vm/drop_caches 
1
```

```shell
echo 1 > /proc/sys/vm/drop_caches
```

1. echo 1 > /proc/sys/vm/drop_caches:表示清除 pagecache
2. echo 2 > /proc/sys/vm/drop_caches:表示清除回收 slab 分配器中的对象(包括目录项缓存和 inode 缓存). slab 分配器是内核中管理内存的一种机制,其中很多缓存数据实现都是用的 pagecache
3. echo 3 > /proc/sys/vm/drop_caches:表示清除 pagecache 和 slab 分配器中的缓存对象

cache不能被回收的情况
===
## tmpfs
```shell
mkdir /tmp/tmpfs
mount -t tmpfs -o size=2G none /tmp/tmpfs 
```

## 共享内存
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <string.h>

#define MEMSIZE 2048*1024*1023

int
main()
{
    int shmid;
    char *ptr;
    pid_t pid;
    struct shmid_ds buf;
    int ret;

    shmid = shmget(IPC_PRIVATE, MEMSIZE, 0600);
    if (shmid<0) {
        perror("shmget()");
        exit(1);
    }

    ret = shmctl(shmid, IPC_STAT, &buf);
    if (ret < 0) {
        perror("shmctl()");
        exit(1);
    }

    printf("shmid: %d\n", shmid);
    printf("shmsize: %d\n", buf.shm_segsz);

    buf.shm_segsz *= 2;

    ret = shmctl(shmid, IPC_SET, &buf);
    if (ret < 0) {
        perror("shmctl()");
        exit(1);
    }

    ret = shmctl(shmid, IPC_SET, &buf);
    if (ret < 0) {
        perror("shmctl()");
        exit(1);
    }

    printf("shmid: %d\n", shmid);
    printf("shmsize: %d\n", buf.shm_segsz);


    pid = fork();
    if (pid<0) {
        perror("fork()");
        exit(1);
    }
    if (pid==0) {
        ptr = shmat(shmid, NULL, 0);
        if (ptr==(void*)-1) {
            perror("shmat()");
            exit(1);
        }
        bzero(ptr, MEMSIZE);
        strcpy(ptr, "Hello!");
        exit(0);
    } else {
        wait(NULL);
        ptr = shmat(shmid, NULL, 0);
        if (ptr==(void*)-1) {
            perror("shmat()");
            exit(1);
        }
        puts(ptr);
        exit(0);
    }
}
```
运行以上c程序,会分配一个2G的共享内存,且程序重没有显示调用shmctl(),所以程序结束共享内存不释放



**删除方法有两种:1. 程序中使用 shmctl()去 IPC_RMID; 2. ipcrm 命令**

### 使用ipcrm命令删除共享内存
```shell
#查询共享内存
ipcs -m

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status      
0x00000000 32768      root       600        2145386496 0 

#命令删除共享内存
ipcrm -m 32768
```

**内核底层在实现共享内存(shm),消息队列(msg)和信号量数组(sem)这些 POSIX:XSI 的 IPC 机制的内存存储时,使用的都是 tmpfs**

## mmap
mmap 就是将一个文件映射进进程的虚拟内存地址,之后就可以通过操作内存的方式对文件的内容进行操作
当 malloc 申请内存时,小段内存内核使用 sbrk 处理,而大段内存就会使用 mmap .当系统调用 exec 族函数执行时,因为其本质上是将一个可执行文件加载到内存执行,所以内核很自然的就可以使用 mmap 方式进行处理

```c
#include <stdlib.h>
#include <stdio.h>
#include <strings.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <fcntl.h>
#include <unistd.h>

#define MEMSIZE 1024*1024*1023*2
#define MPFILE "./mmapfile"

int main()
{
  void *ptr;
  int fd;

  fd = open(MPFILE, O_RDWR);
  if (fd < 0) {
    perror("open()");
    exit(1);
  }

  ptr = mmap(NULL, MEMSIZE, PROT_READ|PROT_WRITE, MAP_SHARED|MAP_ANON, fd, 0);
  if (ptr == NULL) {
    perror("malloc()");
    exit(1);
  }

  printf("%p\n", ptr);
  bzero(ptr, MEMSIZE);

  sleep(100);

  munmap(ptr, MEMSIZE);
  close(fd);

  exit(1);
}
```

```
dd if=/dev/zero of=mmapfile bs=1G count=2
```

程序执行结束后,mmap以 MAP_SHARED 申请的空间自动释放了
