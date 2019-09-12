---
layout: post
title:
modified:
categories: Tech

tags: [libevent, backend]

comments: true
---

<!-- TOC -->

- [signal](#signal)
- [evsig_init](#evsig_init)
- [evsig_cb](#evsig_cb)
- [evmap_signal_active](#evmap_signal_active)
- [event_active_nolock](#event_active_nolock)

<!-- /TOC -->

### signal

signal 机制是 unix 的典型的异步通知系统。注册的 signal 发生时，引起相应的 signal_handler.
libevent 在注册时，可以选择 event type 为 signal，如何做到相容?

本质上是创建一个 socket pair,注册一个 internal 类型的 event，**属性为读和持久，监听的 fd 正是 socketpair 的读端，在 sigal_handler 到来时，往 socket pair 写**，即通过 internal event 注册的 callback 来根据信号值处理不同的信号。

### evsig_init

在每个 IO 复用方法初始化里，都需要调用这个初始化，它就是为 signal 准备的。
注册了 signal 发生时对应 event 的 callback,`evsig_cb`

```c
int
evsig_init(struct event_base *base)
{
	/*
	 * Our signal handler is going to write to one end of the socket
	 * pair to wake up our event loop.  The event loop then scans for
	 * signals that got delivered.
	 */
	//创建socket pair
	if (evutil_socketpair(
		    AF_UNIX, SOCK_STREAM, 0, base->sig.ev_signal_pair) == -1) {
#ifdef WIN32
		/* Make this nonfatal on win32, where sometimes people
		   have localhost firewalled. */
		event_sock_warn(-1, "%s: socketpair", __func__);
#else
		event_sock_err(1, -1, "%s: socketpair", __func__);
#endif
		return -1;
	}

	evutil_make_socket_closeonexec(base->sig.ev_signal_pair[0]);
	evutil_make_socket_closeonexec(base->sig.ev_signal_pair[1]);
	base->sig.sh_old = NULL;
	base->sig.sh_old_max = 0;

	evutil_make_socket_nonblocking(base->sig.ev_signal_pair[0]);
	evutil_make_socket_nonblocking(base->sig.ev_signal_pair[1]);

	//监听 pair[1]的fd,
	event_assign(&base->sig.ev_signal, base, base->sig.ev_signal_pair[1],
		EV_READ | EV_PERSIST, evsig_cb, base);

	base->sig.ev_signal.ev_flags |= EVLIST_INTERNAL;
	//该event 优先级为0,最高级别
	event_priority_set(&base->sig.ev_signal, 0);

	//注册信号的ops
	//添加信号和IO复用等的添加方式当然不同
	base->evsigsel = &evsigops;

	return 0;
}

```

### evsig_cb

在 cb 里记录哪些信号发生了，调用`evmap_signal_active`去 signal active 队列里
执行真正的回调。

```c
static void
evsig_cb(evutil_socket_t fd, short what, void *arg)
{
	static char signals[1024];
	ev_ssize_t n;
	int i;
	int ncaught[NSIG];
	struct event_base *base;

	base = arg;

	memset(&ncaught, 0, sizeof(ncaught));

	while (1) {
		n = recv(fd, signals, sizeof(signals), 0);
		if (n == -1) {
			int err = evutil_socket_geterror(fd);
			if (! EVUTIL_ERR_RW_RETRIABLE(err))
				event_sock_err(1, fd, "%s: recv", __func__);
			break;
		} else if (n == 0) {
			/* XXX warn? */
			break;
		}
		//signals 表示已发生的信号的值
		//在一个callback里，可能１个signal 发生了多次,
		//用ncaught记录次数
		for (i = 0; i < n; ++i) {
			ev_uint8_t sig = signals[i];
			if (sig < NSIG)
				ncaught[sig]++;
		}
	}

	EVBASE_ACQUIRE_LOCK(base, th_base_lock);
	//对已经caught的signal，调用evmap_signal_active
	for (i = 0; i < NSIG; ++i) {
		if (ncaught[i])
			evmap_signal_active(base, i, ncaught[i]);
	}
	EVBASE_RELEASE_LOCK(base, th_base_lock);
}

```

### evmap_signal_active

以 sig 值作为查找 sigevmap 的索引，找到对应的 event,调用`event_active_nolock`

```c
void
evmap_signal_active(struct event_base *base, evutil_socket_t sig, int ncalls)
{
	struct event_signal_map *map = &base->sigmap;
	struct evmap_signal *ctx;
	struct event *ev;

	EVUTIL_ASSERT(sig < map->nentries);
	GET_SIGNAL_SLOT(ctx, map, sig, evmap_signal);

	TAILQ_FOREACH(ev, &ctx->events, ev_signal_next)
		event_active_nolock(ev, EV_SIGNAL, ncalls);
}
```

### event_active_nolock

将注册了该 sig 对应的 event,添加到 active queue 里.所有的事件在真正发生前，
都是先添加到了 active queue 里，由 base_loop 里的`event_process_active`执行真正的 callback。如何真正执行，请参考[event_base_loop 的分析]({{site.url}}/blog/libevent-timing-machanism))。

```c
void
event_active_nolock(struct event *ev, int res, short ncalls)
{
	struct event_base *base;

	event_debug(("event_active: %p (fd "EV_SOCK_FMT"), res %d, callback %p",
		ev, EV_SOCK_ARG(ev->ev_fd), (int)res, ev->ev_callback));


	/* We get different kinds of events, add them together */
	//active时遇到的并不是timeout激活的，设置timeout FLAG并返回
	if (ev->ev_flags & EVLIST_ACTIVE) {
		ev->ev_res |= res;
		return;
	}

	base = ev->ev_base;

	EVENT_BASE_ASSERT_LOCKED(base);

	ev->ev_res = res;

	//优先级不够，要等待
	if (ev->ev_pri < base->event_running_priority)
		base->event_continue = 1;

	if (ev->ev_events & EV_SIGNAL) {
#ifndef _EVENT_DISABLE_THREAD_SUPPORT
		if (base->current_event == ev && !EVBASE_IN_THREAD(base)) {
			++base->current_event_waiters;
			EVTHREAD_COND_WAIT(base->current_event_cond, base->th_base_lock);
		}
#endif
		ev->ev_ncalls = ncalls;
		ev->ev_pncalls = NULL;
	}

	event_queue_insert(base, ev, EVLIST_ACTIVE);

	//非主线程中调用时，通过socketpair通过一个激活一个固定的event
	//唤醒base_loop
	if (EVBASE_NEED_NOTIFY(base))
		evthread_notify_base(base);
}

```
