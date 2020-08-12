---
title: "Linux内核Select和Poll实现"
date: 2020-01-01T15:59:10+08:00
tags: ["linux", "kernel"]
categories: ["linux"]
---

> 等待队列是linux内核中重要的数据结构，许多内核中的功能都依赖其实现，比如异步通知等功能。

<!--more-->

# 一 简介

## 1.1 相关文件

```
kernel/fs/select.c
kernel/include/linux/poll.h
kernel/include/linux/fs.h
kernel/include/linux/sched.h
kernel/include/linux/wait.h
kernel/kernel/sched/wait.c
kernel/kernel/sched/core.c
```

# 二 select
select通过遍历socket数组，并调用对应的设备的poll函数来查询是否有可用资源，如果没有则睡眠。

## 2.1 fd_set
相关结构
```c
#include <sys/select.h>

#define FD_SETSIZE 1024
#define NFDBITS (8 * sizeof(unsigned long))
#define __FDSET_LONGS (FD_SETSIZE/NFDBITS)

typedef struct {
        unsigned long fds_bits[__FDSET_LONGS];
} fd_set;

void FD_SET(int fd, fd_set *fdset)   //将fd添加到fdset
void FD_CLR(int fd, fd_set *fdset)   //从fdset中删除fd
void FD_ISSET(int fd, fd_set *fdset) //判断fd是否已存在fdset
void FD_ZERO(fd_set *fdset)          //初始化fdset内容全为0
```

fd_set是文件描述符的集合，由于每个建成可以打开的文件描述符的的默认数量为1024，fd_set集合的上限默认设定为1024个。

## 2.2 sys_select
select系统调用对应额方法是sys_select，具体代码为


### 1.1.1 select调用堆栈 poll_wait 流程
```c
SYSCALL_DEFINE6(pselect6 ...) at linux/fs/select.c
do_pselect at linux/fs/select.c 
core_sys_select at linux/fs/select.c
do_select at linux/fs/select.c
vfs_poll at linux/fs/select.c
file_operations::poll at linux/include/linux/fs.h
sock_poll at linux/net/socket.c
proto_ops::poll at linux/include/linux/net.h
tcp_poll at linux/net/ipv4/tcp.c
sock_poll_wait at linux/include/net/sock.h
poll_wait at linux/include/linux/poll.h
```

### 1.1.2 select调用堆栈 poll_wake 流程






# 三 总结