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

### 1.1.1 select调用堆栈
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

### 1.1.2 socket write 流程




# 三 总结