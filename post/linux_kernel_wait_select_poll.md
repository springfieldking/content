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
``` c
SYSCALL_DEFINE5(select, int, n, fd_set __user *, inp, fd_set __user *, outp,
		fd_set __user *, exp, struct timeval __user *, tvp)
{
	return kern_select(n, inp, outp, exp, tvp);
}

static int kern_select(int n, fd_set __user *inp, fd_set __user *outp,
		       fd_set __user *exp, struct timeval __user *tvp)
{
	struct timespec64 end_time, *to = NULL;
	struct timeval tv;
	int ret;
	if (tvp) {
		if (copy_from_user(&tv, tvp, sizeof(tv)))
			return -EFAULT;
		to = &end_time;
		if (poll_select_set_timeout(to,
				tv.tv_sec + (tv.tv_usec / USEC_PER_SEC),
				(tv.tv_usec % USEC_PER_SEC) * NSEC_PER_USEC))
			return -EINVAL;
	}
	ret = core_sys_select(n, inp, outp, exp, to);
	ret = poll_select_copy_remaining(&end_time, tvp, PT_TIMEVAL, ret);
	return ret;
}
```

## 2.3 core_sys_select
select方法核心工作:
- 将监控的描述符的事件从用户空间的inp/outp/exp拷到内核空间的in/out/ex
- 调用do_select方法，将in/out/ex监控的事件结果写入到res_in/res_out/res_ex
- 将内核空间的fds的res_in/res_out/res_ex拷贝回用户空间inp/outp/exp

``` c
int core_sys_select(int n, fd_set __user *inp, fd_set __user *outp,
			   fd_set __user *exp, struct timespec64 *end_time)
{
	fd_set_bits fds;
	void *bits;
	int ret, max_fds;
	size_t size, alloc_size;
	struct fdtable *fdt;
	/* Allocate small arguments on the stack to save memory and be faster */
	long stack_fds[SELECT_STACK_ALLOC/sizeof(long)];
	ret = -EINVAL;
	if (n < 0)
		goto out_nofds;
	/* max_fds can increase, so grab it once to avoid race */
	rcu_read_lock();
	fdt = files_fdtable(current->files);
	max_fds = fdt->max_fds;
	rcu_read_unlock();
	if (n > max_fds)
		n = max_fds; // select可监控的个数应当小于等于进程可打开的文件描述符上限
	
	// 计算size=4*(n+32-1)/32
	size = FDS_BYTES(n);
	bits = stack_fds;
	if (size > sizeof(stack_fds) / 6) {
		/* Not enough space in on-stack array; must use kmalloc */
		ret = -ENOMEM;
		if (size > (SIZE_MAX / 6))
			goto out_nofds;
		alloc_size = 6 * size;
		bits = kvmalloc(alloc_size, GFP_KERNEL);
		if (!bits)
			goto out_nofds;
	}
	fds.in      = bits;
	fds.out     = bits +   size;
	fds.ex      = bits + 2*size;
	fds.res_in  = bits + 3*size;
	fds.res_out = bits + 4*size;
	fds.res_ex  = bits + 5*size;
	// 将用户空间的inp/outp/exp拷贝到内核空间的fds的in/out/ex
	if ((ret = get_fd_set(n, inp, fds.in)) ||
	    (ret = get_fd_set(n, outp, fds.out)) ||
	    (ret = get_fd_set(n, exp, fds.ex)))
		goto out;
	zero_fd_set(n, fds.res_in);
	zero_fd_set(n, fds.res_out);
	zero_fd_set(n, fds.res_ex);
	// 执行do_select
	ret = do_select(n, &fds, end_time);
	if (ret < 0)
		goto out;
	if (!ret) {
		ret = -ERESTARTNOHAND;
		if (signal_pending(current))
			goto out;
		ret = 0;
	}
	if (set_fd_set(n, inp, fds.res_in) ||
	    set_fd_set(n, outp, fds.res_out) ||
	    set_fd_set(n, exp, fds.res_ex))
		ret = -EFAULT;
out:
	if (bits != stack_fds)
		kvfree(bits);
out_nofds:
	return ret;
}
```

## 2.4 do_select
do_select核心是通过vfs_poll调用file_operations::poll来检测io事件
- 如果监控事件发生，则将其fd记录下来，退出循环，返回用户空间
- 如果没有事件并且已经超时或者有待处理的信号，也会退出循环，返回用户空间
- 如果都不满足，则让出cpu睡眠，等待事件或者超时。
```c
/*
 * Structures and helpers for select/poll syscall
 */
struct poll_wqueues {
	poll_table pt;
	struct poll_table_page *table;
	struct task_struct *polling_task;
	int triggered;
	int error;
	int inline_index;
	struct poll_table_entry inline_entries[N_INLINE_POLL_ENTRIES];
};

struct poll_table_page {
	struct poll_table_page * next;
	struct poll_table_entry * entry;
	struct poll_table_entry entries[0];
};

struct poll_table_entry {
	struct file *filp;
	__poll_t key;
	wait_queue_entry_t wait;
	wait_queue_head_t *wait_address;
};

/*
 * Do not touch the structure directly, use the access functions
 * poll_does_not_wait() and poll_requested_events() instead.
 */
typedef struct poll_table_struct {
	poll_queue_proc _qproc;
	__poll_t _key;
} poll_table;

static int do_select(int n, fd_set_bits *fds, struct timespec64 *end_time)
{
	ktime_t expire, *to = NULL;
	struct poll_wqueues table; // select等待的队列
	poll_table *wait;
	int retval, i, timed_out = 0;
	u64 slack = 0;
	__poll_t busy_flag = net_busy_loop_on() ? POLL_BUSY_LOOP : 0;
	unsigned long busy_start = 0;
	rcu_read_lock();
	retval = max_select_fd(n, fds);
	rcu_read_unlock();
	if (retval < 0)
		return retval;
	n = retval;
	poll_initwait(&table); // 初始化poll函数指针
	wait = &table.pt;
	if (end_time && !end_time->tv_sec && !end_time->tv_nsec) {
		wait->_qproc = NULL;
		timed_out = 1;
	}
	if (end_time && !timed_out)
		slack = select_estimate_accuracy(end_time);
	retval = 0;
	for (;;) {
		unsigned long *rinp, *routp, *rexp, *inp, *outp, *exp;
		bool can_busy_loop = false;
		inp = fds->in; outp = fds->out; exp = fds->ex;
		rinp = fds->res_in; routp = fds->res_out; rexp = fds->res_ex;
		for (i = 0; i < n; ++rinp, ++routp, ++rexp) {
			unsigned long in, out, ex, all_bits, bit = 1, j;
			unsigned long res_in = 0, res_out = 0, res_ex = 0;
			__poll_t mask;
			in = *inp++; out = *outp++; ex = *exp++;
			all_bits = in | out | ex;
			if (all_bits == 0) {
				i += BITS_PER_LONG;	//以32bits步长遍历位图，直到在该区间存在目标fd
				continue;
			}
			for (j = 0; j < BITS_PER_LONG; ++j, ++i, bit <<= 1) {
				struct fd f;
				if (i >= n)
					break;
				if (!(bit & all_bits))
					continue;
				f = fdget(i); //找到目标fd
				if (f.file) {
					wait_key_set(wait, in, out, bit,
						     busy_flag);
					 //执行文件系统的poll函数，检测IO事件， file_operations::poll at linux/include/linux/fs.h
					mask = vfs_poll(f.file, wait);
					fdput(f);
					//写入in/out/ex相对应的结果
					if ((mask & POLLIN_SET) && (in & bit)) {
						res_in |= bit;
						retval++;
						wait->_qproc = NULL;
					}
					if ((mask & POLLOUT_SET) && (out & bit)) {
						res_out |= bit;
						retval++;
						wait->_qproc = NULL;
					}
					if ((mask & POLLEX_SET) && (ex & bit)) {
						res_ex |= bit;
						retval++;
						wait->_qproc = NULL;
					}
					/* got something, stop busy polling
					 * 如果有结果，则停止轮询
					 */
					if (retval) {
						can_busy_loop = false;
						busy_flag = 0;
					/*
					 * only remember a returned
					 * POLL_BUSY_LOOP if we asked for it
					 */
					} else if (busy_flag & mask)
						can_busy_loop = true;
				}
			}
			// 本轮循环完成，则更新fd事件结果
			if (res_in)
				*rinp = res_in;
			if (res_out)
				*routp = res_out;
			if (res_ex)
				*rexp = res_ex;
			cond_resched(); // 发起调度，让出cpu
		}
		wait->_qproc = NULL;
		// 当有文件描述符准备就绪 或者超时 或者 有待处理的信号，则退出循环
		if (retval || timed_out || signal_pending(current))
			break;
		if (table.error) {
			retval = table.error;
			break;
		}
		/* only if found POLL_BUSY_LOOP sockets && not out of time */
		if (can_busy_loop && !need_resched()) {
			if (!busy_start) {
				busy_start = busy_loop_current_time();
				continue;
			}
			if (!busy_loop_timeout(busy_start))
				continue;
		}
		busy_flag = 0;
		/*
		 * 首轮循环超时设置
		 */
		if (end_time && !to) {
			expire = timespec64_to_ktime(*end_time);
			to = &expire;
		}
		// 设置当前进程状态为TASK_INTERRUPTIBLE，进入睡眠直到超时
		if (!poll_schedule_timeout(&table, TASK_INTERRUPTIBLE,
					   to, slack))
			timed_out = 1;
	}
	poll_freewait(&table);
	return retval;
}

```


### 2.4.1 __pollwait
上面提到do_select会调用file_operations::poll检测事件，除此之外还是调用__poitwait函数，设置唤醒的为pollwake
调用堆栈为:
```c
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
     在tcp_poll中检测各种事件
```

``` c
void poll_initwait(struct poll_wqueues *pwq)
{
	init_poll_funcptr(&pwq->pt, __pollwait); //初始化poll函数指针
	pwq->polling_task = current;
	pwq->triggered = 0;
	pwq->error = 0;
	pwq->table = NULL;
	pwq->inline_index = 0;
}
static inline void init_poll_funcptr(poll_table *pt, poll_queue_proc qproc)
{
	pt->_qproc = qproc;
	pt->_key   = ~(__poll_t)0; /* all events enabled */
}
/* Add a new entry */
static void __pollwait(struct file *filp, wait_queue_head_t *wait_address,
				poll_table *p)
{
	struct poll_wqueues *pwq = container_of(p, struct poll_wqueues, pt);
	struct poll_table_entry *entry = poll_get_entry(pwq);
	if (!entry)
		return;
	entry->filp = get_file(filp);
	entry->wait_address = wait_address;
	entry->key = p->_key;
	init_waitqueue_func_entry(&entry->wait, pollwake);
	entry->wait.private = pwq;
	add_wait_queue(wait_address, &entry->wait);
}

```

### 2.4.2 poll_schedule_timeout
设置睡眠超时等待时间
```c
static int poll_schedule_timeout(struct poll_wqueues *pwq, int state,
			  ktime_t *expires, unsigned long slack)
{
	int rc = -EINTR;
	set_current_state(state); 	//设置进程状态
	if (!pwq->triggered)		//设置超时
		rc = schedule_hrtimeout_range(expires, slack, HRTIMER_MODE_ABS);
	__set_current_state(TASK_RUNNING);
	smp_store_mb(pwq->triggered, 0);
	return rc;
}

```


### 2.4.3 pollwake
```
static int pollwake(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
{
	struct poll_table_entry *entry;
	entry = container_of(wait, struct poll_table_entry, wait);
	if (key && !(key_to_poll(key) & entry->key))
		return 0;
	return __pollwake(wait, mode, sync, key);
}
static int __pollwake(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
{
	struct poll_wqueues *pwq = wait->private;
	DECLARE_WAITQUEUE(dummy_wait, pwq->polling_task);
	smp_wmb();
	pwq->triggered = 1;
	return default_wake_function(&dummy_wait, mode, sync, key);
}

```

调用堆栈

``` c
  0xffffffff8a2f4d10 : pollwake+0x0/0x90 [kernel]
 0xffffffff8a0f48ee : __wake_up_common+0x7e/0x140 [kernel]
 0xffffffff8a0f4a2c : __wake_up_common_lock+0x7c/0xc0 [kernel]
 0xffffffff8a0f4cae : __wake_up_sync_key+0x1e/0x30 [kernel]
 0xffffffff8a914720 : sock_def_readable+0x40/0x70 [kernel]
 0xffffffff8a9ccb28 : tcp_data_ready+0x48/0x50 [kernel]
 0xffffffff8a9ce802 : tcp_rcv_established+0x5a2/0x650 [kernel]
 0xffffffff8a9db3e0 : tcp_v4_do_rcv+0x140/0x200 [kernel]
 0xffffffff8a9dcfd1 : tcp_v4_rcv+0xbf1/0xd00 [kernel]
 0xffffffff8a9af680 : ip_protocol_deliver_rcu+0x30/0x1b0 [kernel]
 0xffffffff8a9af848 : ip_local_deliver_finish+0x48/0x50 [kernel]
 0xffffffff8a9af8c3 : ip_local_deliver+0x73/0xf0 [kernel]
 0xffffffff8a9aefe5 : ip_rcv_finish+0x85/0xa0 [kernel]
 0xffffffff8a9af9fc : ip_rcv+0xbc/0xd0 [kernel]
 0xffffffff8a935d78 : __netif_receive_skb_one_core+0x88/0xa0 [kernel]
 0xffffffff8a935dc8 : __netif_receive_skb+0x18/0x60 [kernel]
 0xffffffff8a935e55 : netif_receive_skb_internal+0x45/0xc0 [kernel]
 0xffffffff8a9378af : napi_gro_receive+0xff/0x160 [kernel]
 0xffffffffc03ce563
 0xffffffffc03ce563
 0xffffffffc03ceea0
 0xffffffff8a936eda : net_rx_action+0x13a/0x370 [kernel]
 0xffffffff8ae000e1 : __do_softirq+0xe1/0x2d6 [kernel]
 0xffffffff8a0a81ce : irq_exit+0xae/0xb0 [kernel]
 0xffffffff8ac01e7a : do_IRQ+0x5a/0xf0 [kernel]
 0xffffffff8ac00a0f : ret_from_intr+0x0/0x1d [kernel]
 0xffffffff8aad564e : native_safe_halt+0xe/0x10 [kernel]
 0xffffffff8a03dd55 : arch_cpu_idle+0x15/0x20 [kernel]
 0xffffffff8aad57f3 : default_idle_call+0x23/0x30 [kernel]
 0xffffffff8a0e006b : do_idle+0x1fb/0x270 [kernel]
 0xffffffff8a0e0270 : cpu_startup_entry+0x20/0x30 [kernel]
 0xffffffff8a064447 : start_secondary+0x167/0x1c0 [kernel]
 0xffffffff8a0000d4 : secondary_startup_64+0xa4/0xb0 [kernel]
```

# 2.5 select 小结




# 三 总结