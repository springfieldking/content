---
title: "Linux内核等待队列"
date: 2020-01-01T15:59:10+08:00
tags: ["linux", "kernel"]
categories: ["linux"]
---

> 等待队列是linux内核中重要的数据结构，许多内核中的功能都依赖其实现，比如异步通知等功能。

<!--more-->

# 一 简介

## 1.1 相关文件

```
kernel/include/linux/wait.h
kernel/kernel/sched/wait.c
kernel/include/linux/sched.h
kernel/kernel/sched/core.c
```

## 1.2 相关数据结构

```c
/*
 * A single wait-queue entry structure:
 */
struct wait_queue_entry {
	unsigned int		flags;
	void			    *private;
	wait_queue_func_t	func;
	struct list_head	entry;
};
struct wait_queue_head {
	spinlock_t		    lock;
	struct list_head	head;
};
typedef struct wait_queue_head wait_queue_head_t;
```

## 1.3 相关方法

### 1.3.1 声明创建
```c
#define __WAITQUEUE_INITIALIZER(name, tsk) {					\
	.private	= tsk,							\
	.func		= default_wake_function,				\
	.entry		= { NULL, NULL } }

#define DECLARE_WAITQUEUE(name, tsk)						\
	struct wait_queue_entry name = __WAITQUEUE_INITIALIZER(name, tsk)

#define __WAIT_QUEUE_HEAD_INITIALIZER(name) {					\
	.lock		= __SPIN_LOCK_UNLOCKED(name.lock),			\
	.head		= { &(name).head, &(name).head } }

#define DECLARE_WAIT_QUEUE_HEAD(name) \
	struct wait_queue_head name = __WAIT_QUEUE_HEAD_INITIALIZER(name)

```

### 1.3.2 添加到队列
```c
void add_wait_queue(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry)
{
	unsigned long flags;
	wq_entry->flags &= ~WQ_FLAG_EXCLUSIVE;
	spin_lock_irqsave(&wq_head->lock, flags);
	__add_wait_queue(wq_head, wq_entry);
	spin_unlock_irqrestore(&wq_head->lock, flags);
}

static inline void __add_wait_queue(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry)
{
	list_add(&wq_entry->entry, &wq_head->head);
}

```

### 1.3.3 从队列移除

```c
void remove_wait_queue(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry)
{
	unsigned long flags;
	spin_lock_irqsave(&wq_head->lock, flags);
	__remove_wait_queue(wq_head, wq_entry);
	spin_unlock_irqrestore(&wq_head->lock, flags);
}


static inline void
__remove_wait_queue(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry)
{
	list_del(&wq_entry->entry);
}

```


# 二 休眠与唤醒

## 2.1 等待

```c
#define wait_event(wq_head, condition)						\
do {										\
	might_sleep();								\
	if (condition)								\
		break;								\
	__wait_event(wq_head, condition);					\
} while (0)

#define __wait_event(wq_head, condition)					\
	(void)___wait_event(wq_head, condition, TASK_UNINTERRUPTIBLE, 0, 0,	\
			    schedule())

#define ___wait_event(wq_head, condition, state, exclusive, ret, cmd)		\
({										\
	__label__ __out;							\
	struct wait_queue_entry __wq_entry;					\
	long __ret = ret;	/* explicit shadow */				\
										\
	init_wait_entry(&__wq_entry, exclusive ? WQ_FLAG_EXCLUSIVE : 0);	\
	for (;;) {								\
		// 检测当前进程是否待处理信号则返回__int非0 \
		long __int = prepare_to_wait_event(&wq_head, &__wq_entry, state);\
										\
		if (condition)							\
			break;							\
		// 当有待处理信号且进程处于中断状态(TASK_INTERRUPTIBLE或TASK_KILLABLE)则跳出等待循环 \
		if (___wait_is_interruptible(state) && __int) {			\
			__ret = __int;						\
			goto __out;						\
		}								\
		// 执行schedule()进入睡眠			\
		cmd;								\
	}									\
	finish_wait(&wq_head, &__wq_entry);					\
__out:	__ret;									\
})

```

### 2.1.1 prepare_to_wait_event

```c
long prepare_to_wait_event(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry, int state)
{
	unsigned long flags;
	long ret = 0;
	spin_lock_irqsave(&wq_head->lock, flags);
	if (signal_pending_state(state, current)) {	// 信号检测
		list_del_init(&wq_entry->entry);
		ret = -ERESTARTSYS;
	} else {
		if (list_empty(&wq_entry->entry)) {
			if (wq_entry->flags & WQ_FLAG_EXCLUSIVE)
				__add_wait_queue_entry_tail(wq_head, wq_entry);
			else
				__add_wait_queue(wq_head, wq_entry);
		}
		set_current_state(state);
	}
	spin_unlock_irqrestore(&wq_head->lock, flags);
	return ret;
}
```

### 2.1.2 autoremove_wake_function
```c
int autoremove_wake_function(struct wait_queue_entry *wq_entry, unsigned mode, int sync, void *key)
{
	int ret = default_wake_function(wq_entry, mode, sync, key);
	if (ret)
		list_del_init(&wq_entry->entry);
	return ret;
}
```


## 2.2 唤醒
```c
#define wake_up(x)			__wake_up(x, TASK_NORMAL, 1, NULL)

void __wake_up(struct wait_queue_head *wq_head, unsigned int mode,
			int nr_exclusive, void *key)
{
	__wake_up_common_lock(wq_head, mode, nr_exclusive, 0, key);
}

static void __wake_up_common_lock(struct wait_queue_head *wq_head, unsigned int mode,
			int nr_exclusive, int wake_flags, void *key)
{
	unsigned long flags;
	wait_queue_entry_t bookmark;
	bookmark.flags = 0;
	bookmark.private = NULL;
	bookmark.func = NULL;
	INIT_LIST_HEAD(&bookmark.entry);
	spin_lock_irqsave(&wq_head->lock, flags);
	nr_exclusive = __wake_up_common(wq_head, mode, nr_exclusive, wake_flags, key, &bookmark);
	spin_unlock_irqrestore(&wq_head->lock, flags);
	while (bookmark.flags & WQ_FLAG_BOOKMARK) {
		spin_lock_irqsave(&wq_head->lock, flags);
		nr_exclusive = __wake_up_common(wq_head, mode, nr_exclusive,
						wake_flags, key, &bookmark);
		spin_unlock_irqrestore(&wq_head->lock, flags);
	}
}

```

### 2.2.1 default_wake_function
```c
int default_wake_function(wait_queue_entry_t *curr, unsigned mode, int wake_flags,
			  void *key)
{
	return try_to_wake_up(curr->private, mode, wake_flags);
}
```

### 2.2.2 try_to_wake_up
```c
try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)
{
    unsigned long flags;
    int cpu, src_cpu, success = 0;

    bool freq_notif_allowed = !(wake_flags & WF_NO_NOTIFIER);
    bool check_group = false;
    wake_flags &= ~WF_NO_NOTIFIER;

    smp_mb__before_spinlock();
    raw_spin_lock_irqsave(&p->pi_lock, flags); //关闭本地中断
    src_cpu = cpu = task_cpu(p);

    //如果当前进程状态不属于可唤醒状态集，则无法唤醒该进程
    //wake_up()传递过来的TASK_NORMAL等于(TASK_INTERRUPTIBLE | TASK_UNINTERRUPTIBLE)
    if (!(p->state & state)) 
        goto out;

    success = 1; 
    smp_rmb();
    if (p->on_rq && ttwu_remote(p, wake_flags)) //当前进程已处于rq运行队列，则无需唤醒
        goto stat;
    /*...*/
    
    ttwu_queue(p, cpu); //【小节3.2.3】
stat:
    ttwu_stat(p, cpu, wake_flags);
out:
    raw_spin_unlock_irqrestore(&p->pi_lock, flags); //恢复本地中断
    /*...*/
    return success;
}
```

### 2.2.3 ttwu_queue

```c
static void ttwu_queue_remote(struct task_struct *p, int cpu, int wake_flags)
{
	struct rq *rq = cpu_rq(cpu);
	p->sched_remote_wakeup = !!(wake_flags & WF_MIGRATED);
	if (llist_add(&p->wake_entry, &cpu_rq(cpu)->wake_list)) {
		if (!set_nr_if_polling(rq->idle))
			smp_send_reschedule(cpu);
		else
			trace_sched_wake_idle_without_ipi(cpu);
	}
}
```


### 2.2.4 ttwu_do_activate
```c
static void
ttwu_do_activate(struct rq *rq, struct task_struct *p, int wake_flags,
		 struct rq_flags *rf)
{
	int en_flags = ENQUEUE_WAKEUP | ENQUEUE_NOCLOCK;
	lockdep_assert_held(&rq->lock);
#ifdef CONFIG_SMP
	if (p->sched_contributes_to_load)
		rq->nr_uninterruptible--;
	if (wake_flags & WF_MIGRATED)
		en_flags |= ENQUEUE_MIGRATED;
#endif
	ttwu_activate(rq, p, en_flags);
	ttwu_do_wakeup(rq, p, wake_flags, rf);
}
```

# 三 总结