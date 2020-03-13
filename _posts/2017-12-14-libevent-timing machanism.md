---
layout: post
title:
modified:
categories: Tech

tags: [libevent, backend]
---

<!-- TOC -->

- [分析](#分析)
- [evene_base_loop](#evene_base_loop)
- [event_process_active_single_queue](#event_process_active_single_queue)
- [event_signal_closure](#event_signal_closure)
- [event_persist_closure](#event_persist_closure)

<!-- /TOC -->

### 分析

超时应该是 libevent 封装的最核心功能之一,也可以理解为定时器的实现原理。为了执行回调，libevent 用了队列的数据结构来缓冲 callback

核心流程在`event_base_loop`函数中实现。

- 1
  在 event_base_loop 中，必然有个顶层的 while,每次跳出 IO 复用时,需要刷新 IO 复用的等待时间(如 select 的超时参数)。
- 2
  每个超时 event 都用绝对值记录了要发生的时间点,比如某个 event READ 超时为相对调用时的 100ms，先要转换成绝对时间，比如 1:00:00,转换成秒就是 3600s。

- 3
  event_base 保存了绝对的时间起点(event_base->event_tv)，有了它才能根据相对时间计算绝对时间，而且需要函数`time_correct`修正。

- 4

相对的超时时间用 min_heap 记录，可以记录非常多的超时时间，可以从堆顶很方便的取出 minimum time.libevent 作者甚至还嫌弃 min_heap 效率不够高，用 common_timeout 来针对不变的时间，为了提高效率真是挖空心思.这看看[这篇文章](http://blog.csdn.net/luotuo44/article/details/38678333)

- 5

把 IO 复用的调用理解为 dispatch,需要计算当前时间(nowtime)和 event 中的最小时间(minimum time)的差值 tv_p,该部分在`time_next`中实现。`tv_p>0`说明最小的超时事件还没发生，仍然需要等待 tv_p 的时间。 如果为 0，不需要超时,则同步阻塞等待。

```cpp
res = evsel->dispatch(base, tv_p);
```

- 6
  dispatch 返回，说明有事件发生。逐个遍历 min_heap 中的每个 time 和当前时间 nowtime 比较，如果`time < now`,则超时已经发生了，需要调用相应的回调`event_active_nolock`,并删除该 event 的事件。直到遇到`time > now`,则停止回调，整个实现在`timeout_process`中。

- 7
  真正的执行 callback 在`event_process_active`。callback 是在已经 insert 的 active 队列里按优先级从队列里逐个执行回调。添加到 active 队列是用`event_queue_insert`函数。

整理下１个回调的整个调用链条是:

```sh
event_base_loop-> dispatch->event_process_acitive->callbacks in queue
```

回过头看，还是**IO 复用模式的骨架**。

### evene_base_loop

一些地方用中文做了注解.

```c
event_base_loop(struct event_base *base, int flags)
{
	const struct eventop *evsel = base->evsel;
	struct timeval tv;
	struct timeval *tv_p;
	int res, done, retval = 0;

	/* Grab the lock.  We will release it inside evsel.dispatch, and again
	 * as we invoke user callbacks. */
	EVBASE_ACQUIRE_LOCK(base, th_base_lock);
    //允许一个唯一的loop
	if (base->running_loop) {
		event_warnx("%s: reentrant invocation.  Only one event_base_loop"
		    " can run on each event_base at once.", __func__);
		EVBASE_RELEASE_LOCK(base, th_base_lock);
		return -1;
	}

	base->running_loop = 1;

	clear_time_cache(base);

	if (base->sig.ev_signal_added && base->sig.ev_n_signals_added)
		evsig_set_base(base);

	done = 0;

#ifndef _EVENT_DISABLE_THREAD_SUPPORT
	base->th_owner_id = EVTHREAD_GET_ID();
#endif

	base->event_gotterm = base->event_break = 0;

	while (!done) {
		base->event_continue = 0;

		/* Terminate the loop if we have been asked to */
		if (base->event_gotterm) {
			break;
		}

		if (base->event_break) {
			break;
		}

		timeout_correct(base, &tv);

        tv_p = &tv;
        //如果当前有event激活才计算超时时间
		if (!N_ACTIVE_CALLBACKS(base) && !(flags & EVLOOP_NONBLOCK)) {
			timeout_next(base, &tv_p);
		} else {
			/*
			 * if we have active events, we just poll new events
			 * without waiting.
			 */
			evutil_timerclear(&tv);
		}

        //如果没有event了，则退出loop
		/* If we have no events, we just exit */
		if (!event_haveevents(base) && !N_ACTIVE_CALLBACKS(base)) {
			event_debug(("%s: no events registered.", __func__));
			retval = 1;
			goto done;
		}

        //上面的步骤也是耗时的，需要更新event_base的基准时间
		/* update last old time */
		gettime(base, &base->event_tv);

		clear_time_cache(base);

        //核心的dispatch即IO复用的调用,tv_p很关键
		res = evsel->dispatch(base, tv_p);

		if (res == -1) {
			event_debug(("%s: dispatch returned unsuccessfully.",
				__func__));
			retval = -1;
			goto done;
		}

        //又是为了效率，为了不同每次调用time_api，可以把系统的的时间cache起来
        update_time_cache(base);


        timeout_process(base);

        //从这里激发回调
		if (N_ACTIVE_CALLBACKS(base)) {
			int n = event_process_active(base);
			if ((flags & EVLOOP_ONCE)
			    && N_ACTIVE_CALLBACKS(base) == 0
			    && n != 0)
				done = 1;
		} else if (flags & EVLOOP_NONBLOCK)
			done = 1;
	}
	event_debug(("%s: asked to terminate loop.", __func__));

done:
	clear_time_cache(base);
	base->running_loop = 0;

	EVBASE_RELEASE_LOCK(base, th_base_lock);

	return (retval);
}
```

### event_process_active_single_queue

在`event_process_acitive`被调用，是真正执行回调的地方，

```c
event_process_active_single_queue(struct event_base *base,
    struct event_list *activeq)
{
	struct event *ev;
	int count = 0;

	EVUTIL_ASSERT(activeq != NULL);

	for (ev = TAILQ_FIRST(activeq); ev; ev = TAILQ_FIRST(activeq)) {
		if (ev->ev_events & EV_PERSIST)
			event_queue_remove(base, ev, EVLIST_ACTIVE);
		else
			event_del_internal(ev);
		//该函数可能发生多个callback
		if (!(ev->ev_flags & EVLIST_INTERNAL))
			++count;

		event_debug((
			 "event_process_active: event: %p, %s%scall %p",
			ev,
			ev->ev_res & EV_READ ? "EV_READ " : " ",
			ev->ev_res & EV_WRITE ? "EV_WRITE " : " ",
			ev->ev_callback));

#ifndef _EVENT_DISABLE_THREAD_SUPPORT
		base->current_event = ev;
		base->current_event_waiters = 0;
#endif

		switch (ev->ev_closure) {
		case EV_CLOSURE_SIGNAL:
			event_signal_closure(base, ev);
			break;
		case EV_CLOSURE_PERSIST:
			event_persist_closure(base, ev);
			break;
		default:
		case EV_CLOSURE_NONE:
			EVBASE_RELEASE_LOCK(base, th_base_lock);
			(*ev->ev_callback)(
				ev->ev_fd, ev->ev_res, ev->ev_arg);
			break;
		}

		EVBASE_ACQUIRE_LOCK(base, th_base_lock);
#ifndef _EVENT_DISABLE_THREAD_SUPPORT
		base->current_event = NULL;
		if (base->current_event_waiters) {
			base->current_event_waiters = 0;
			EVTHREAD_COND_BROADCAST(base->current_event_cond);
		}
#endif

		if (base->event_break)
			return -1;
		if (base->event_continue)
			break;
	}
	return count;
}

```

这段代码决定了真正的 callback 调用，又根据 closure 分为了不同的调用执行方式。

```c
switch (ev->ev_closure) {
		case EV_CLOSURE_SIGNAL:
			event_signal_closure(base, ev);
			break;
		case EV_CLOSURE_PERSIST:
			event_persist_closure(base, ev);
			break;
		default:
		case EV_CLOSURE_NONE:
			EVBASE_RELEASE_LOCK(base, th_base_lock);
			(*ev->ev_callback)(
				ev->ev_fd, ev->ev_res, ev->ev_arg);
			break;
		}
```

### event_signal_closure

信号类的回调, ncalls 记录多少个相同的信号同时发生了。
event_break 给了中止同时多个回调执行的机会,由`event_base_loopbreak`控制。

```c
/* "closure" function called when processing active signal events */
static inline void
event_signal_closure(struct event_base *base, struct event *ev)
{
	short ncalls;
	int should_break;

	/* Allows deletes to work */
	ncalls = ev->ev_ncalls;
	if (ncalls != 0)
		ev->ev_pncalls = &ncalls;
	EVBASE_RELEASE_LOCK(base, th_base_lock);
	while (ncalls) {
		ncalls--;
		ev->ev_ncalls = ncalls;
		if (ncalls == 0)
			ev->ev_pncalls = NULL;
		(*ev->ev_callback)(ev->ev_fd, ev->ev_res, ev->ev_arg);

		EVBASE_ACQUIRE_LOCK(base, th_base_lock);
		should_break = base->event_break;
		EVBASE_RELEASE_LOCK(base, th_base_lock);

		if (should_break) {
			if (ncalls != 0)
				ev->ev_pncalls = NULL;
			return;
		}
	}
}

```

### event_persist_closure

event 类型为 EV_PERSIST，这个是持续的触发回到的意思,否则每 add 一次，只触发一次。
在超时事件 event assign 时，如果标记为 EV_PERSIST,则会记住该超时时间在`ev_io_timeout`里。

```
			ev->ev_io_timeout = *tv;
```
