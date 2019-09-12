---
layout: post
title:
modified:
categories: Tech

tags: [libevent, backend]
---

<!-- TOC -->

- [event](#event)
- [event_add](#event_add)
- [添加到数据结构](#添加到数据结构)
- [event_queue_insert](#event_queue_insert)
- [特别的处理](#特别的处理)
- [event_add_internal 代码附录](#event_add_internal-代码附录)

<!-- TOC -->

### event

event 是 libeven reactor 模式中的基本处理单位，所有事件都是基于 event 去添加，事件类型有:

- 普通 IO
- signal 事件
- 定时器
  以上事件都可以设置`超时返回时间`，成为相应的超时事件。
  详细可以看看之前的[这篇文章]({{site.url}}/blog/libevent-timing-machanism)

### event_add

本质调用`event_add_internal`,调用前加了把锁。

1. 只要是超时事件，分配 min_heap 资源

![](http://osvo72tet.bkt.clouddn.com/20170809-112244.png)

2. signal 事件的多线程 add 处理,在 signal hander 时，又要 add 相同事件,则需要排队
   signal 事件和普通 io 的处理是不一样的。

![](http://osvo72tet.bkt.clouddn.com/20170809-112305.png)

3. 以上两种事件分别添加到对应的数据结构里,再将 event 添加到 acitive_queue 里，标记状态为 INSERTED；

![](http://osvo72tet.bkt.clouddn.com/20170808-180752.png)

### 添加到数据结构

- evmap_io_add 普通 io 事件添加

event 按 fd 作为索引，添加到链表数组中. 因为要当 queue 用，先进先出，所以是`insert_tail,remove_head`
如果 fd 没被添加过，通过 eventop（IO 复用模型）的 add 把 fd 添加到 IO 复用的监控列表里。
如果已经是 IO 多路复用已经添加过的 fd(event list 不为空),那就在队尾插入该事件。

![](http://osvo72tet.bkt.clouddn.com/event-io-map.png)

- evmap_signal_add signal 事件的添加

用的数据结构与上面的 evmap_io_add 一致。
[这篇文章讲了](http://blog.csdn.net/liwsustc/article/details/38348263) signal 这种异步机制是怎么纳入框架。sig_hander 里用 socketpair 写，读端用 event 标记为 READ...

### event_queue_insert

设置 queue 状态,然后依据状态把 event 添加到 base 的管理链表中.

![](http://osvo72tet.bkt.clouddn.com/event_queue_insert.png)

```
#define EVLIST_TIMEOUT	0x01
#define EVLIST_INSERTED	0x02
#define EVLIST_SIGNAL	0x04
#define EVLIST_ACTIVE	0x08
#define EVLIST_INTERNAL	0x10
#define EVLIST_INIT	0x80
```

- EVLIST_INSERTED:
  刚添加的事件，在 base 的 event_queue 中
- EVLIST_ACTIVE:
  base 的 active_queues 中，类似 event_io_map 的链表数组，不同优先级索引寻访不同的链表头 active_queues[pri].
- EVLIST_TIMEOUT:
  超时事件，大量的相等时间的超时用 common_timeout_list 管理，否则用 min_heap_push；

### 特别的处理

- n 个 event 可以从多个线程添加,处理多线程
- 等待 event 时，添加 event? io 复用中包括一个专门的内部 event, add 时，向该 fd 发送一个字节，即可唤醒 IO 复用，然后添加新的事件(fd).

### event_add_internal 代码附录

```cpp
static inline int
event_add_internal(struct event *ev, const struct timeval *tv,
    int tv_is_absolute)
{
    struct event_base *base = ev->ev_base;
    int res = 0;
    int notify = 0;

    //锁确认
	EVENT_BASE_ASSERT_LOCKED(base);
	_event_debug_assert_is_setup(ev);

	event_debug((
		 "event_add: event: %p (fd "EV_SOCK_FMT"), %s%s%scall %p",
		 ev,
		 EV_SOCK_ARG(ev->ev_fd),
		 ev->ev_events & EV_READ ? "EV_READ " : " ",
		 ev->ev_events & EV_WRITE ? "EV_WRITE " : " ",
		 tv ? "EV_TIMEOUT " : " ",
		 ev->ev_callback));

EVUTIL_ASSERT(!(ev->ev_flags & ~EVLIST_ALL));

	/*
	 * prepare for timeout insertion further below, if we get a
	 * failure on any step, we should not change any state.
	 */
	 //如果有超时时间tv，而且该event之前没有设置过超时,分配min_heap
	if (tv != NULL && !(ev->ev_flags & EVLIST_TIMEOUT)) {
		if (min_heap_reserve(&base->timeheap,
			1 + min_heap_size(&base->timeheap)) == -1)
			return (-1);  /* ENOMEM == errno */
	}

	/* If the main thread is currently executing a signal event's
	 * callback, and we are not the main thread, then we want to wait
	 * until the callback is done before we mess with the event, or else
	 * we can race on ev_ncalls and ev_pncalls below. */
	 //signal类的多线程event不能在callback期间同时添加，需挂起等待
#ifndef _EVENT_DISABLE_THREAD_SUPPORT
	if (base->current_event == ev && (ev->ev_events & EV_SIGNAL)
	    && !EVBASE_IN_THREAD(base)) {
		++base->current_event_waiters;
		EVTHREAD_COND_WAIT(base->current_event_cond, base->th_base_lock);
	}
#endif

	if ((ev->ev_events & (EV_READ|EV_WRITE|EV_SIGNAL)) &&
	    !(ev->ev_flags & (EVLIST_INSERTED|EVLIST_ACTIVE))) {
		//普通io
		if (ev->ev_events & (EV_READ|EV_WRITE))
			res = evmap_io_add(base, ev->ev_fd, ev);
		else if (ev->ev_events & EV_SIGNAL)
		//signal
			res = evmap_signal_add(base, (int)ev->ev_fd, ev);
		//激活队列
		if (res != -1)
			event_queue_insert(base, ev, EVLIST_INSERTED);
		if (res == 1) {
			/* evmap says we need to notify the main thread. */
			notify = 1;
			res = 0;
		}
	}

	/*
	 * we should change the timeout state only if the previous event
	 * addition succeeded.
	 */
	if (res != -1 && tv != NULL) {
		struct timeval now;
		int common_timeout;

		/*
		 * for persistent timeout events, we remember the
		 * timeout value and re-add the event.
		 *
		 * If tv_is_absolute, this was already set.
		 */
		if (ev->ev_closure == EV_CLOSURE_PERSIST && !tv_is_absolute)
			ev->ev_io_timeout = *tv;

		/*
		 * we already reserved memory above for the case where we
		 * are not replacing an existing timeout.
		 */
		if (ev->ev_flags & EVLIST_TIMEOUT) {
			/* XXX I believe this is needless. */
			if (min_heap_elt_is_top(ev))
				notify = 1;
			event_queue_remove(base, ev, EVLIST_TIMEOUT);
		}

		/* Check if it is active due to a timeout.  Rescheduling
		 * this timeout before the callback can be executed
		 * removes it from the active list. */
		if ((ev->ev_flags & EVLIST_ACTIVE) &&
		    (ev->ev_res & EV_TIMEOUT)) {
			if (ev->ev_events & EV_SIGNAL) {
				/* See if we are just active executing
				 * this event in a loop
				 */
				if (ev->ev_ncalls && ev->ev_pncalls) {
					/* Abort loop */
					*ev->ev_pncalls = 0;
				}
			}

			event_queue_remove(base, ev, EVLIST_ACTIVE);
		}

		gettime(base, &now);

		common_timeout = is_common_timeout(tv, base);
		if (tv_is_absolute) {
			ev->ev_timeout = *tv;
		} else if (common_timeout) {
			struct timeval tmp = *tv;
			tmp.tv_usec &= MICROSECONDS_MASK;
			evutil_timeradd(&now, &tmp, &ev->ev_timeout);
			ev->ev_timeout.tv_usec |=
			    (tv->tv_usec & ~MICROSECONDS_MASK);
		} else {
			evutil_timeradd(&now, tv, &ev->ev_timeout);
		}

		event_debug((
			 "event_add: timeout in %d seconds, call %p",
			 (int)tv->tv_sec, ev->ev_callback));

		event_queue_insert(base, ev, EVLIST_TIMEOUT);
		if (common_timeout) {
			struct common_timeout_list *ctl =
			    get_common_timeout_list(base, &ev->ev_timeout);
			if (ev == TAILQ_FIRST(&ctl->events)) {
				common_timeout_schedule(ctl, &now, ev);
			}
		} else {
			/* See if the earliest timeout is now earlier than it
			 * was before: if so, we will need to tell the main
			 * thread to wake up earlier than it would
			 * otherwise. */
			if (min_heap_elt_is_top(ev))
				notify = 1;
		}
	}

	/* if we are not in the right thread, we need to wake up the loop */
	if (res != -1 && notify && EVBASE_NEED_NOTIFY(base))
		evthread_notify_base(base);

	_event_debug_note_add(ev);

	return (res);
}
```

[参考](http://blog.csdn.net/luotuo44/article/details/38637671)
