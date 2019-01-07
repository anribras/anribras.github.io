---
layout: post
title:
modified:
categories: Tech
 
tags: [libevent,backend]

  
comments: true
---
<!-- TOC -->

- [min_heap管理时间的问题](#min_heap管理时间的问题)
- [解决思路](#解决思路)
- [设置common_timeout](#设置common_timeout)
- [回调 common_timeout_callback](#回调-common_timeout_callback)
- [event_add中的common_timeout](#event_add中的common_timeout)

<!-- /TOC -->

### min_heap管理时间的问题 

推荐[这篇文章](http://blog.csdn.net/luotuo44/article/details/38678333),基于他的理解。

假设有1000k个超时100ms的事件，1000k都是在100ms内添加完成，也就是再第１次超时发生前就完成。如此多是event_add对于min heap其实压力也不小，O(log(N))的复杂度不见得能接受。
 
### 解决思路
因为有`相同时间`的这个特性,考虑把这些event用另外的数据结构(queue array)管理，因为event_add的先后顺序，决定了后面add的肯定是后发生的。仅需要添加一个`内部event`,时间设置为100ms(纯粹的超时监控即可，不需要fd,也就是按常规的放入min_heap),当其发生时，相当于是queue head的event触发了,再去处理队首的event即可。

libevent采用common_timeout_list** 来管理相同的超时时间，也就是是一个队列数组，index表示不同的超时时间,不同的超时event放在不同的commont_timeout_list*，**来缓冲直接放入min_heap的压力**。

具体做法就是，event_add时，如果设置了common_timeout,则会根据设置优先进入common_timeout_list而不是min_heap。但是会把队首的event按正常方式add,
添加１个肯定比直接添加10000个效率高。

### 设置common_timeout
设置这个时长需要手动设置才行，因此这个机制的的使用和应用场景是直接相关,通常并不需要使用。
调用的函数为`event_base_init_common_timeout`。
分配队列数组，并设置好某个时间对应index的队列的首元素，注册`internal event`,这是libevent第2个用到internal event的地方，之前在sigevent处理时用到过一次。
```c
const struct timeval *
event_base_init_common_timeout(struct event_base *base,
    const struct timeval *duration)
{
	int i;
	struct timeval tv;
	const struct timeval *result=NULL;
	struct common_timeout_list *new_ctl;

	EVBASE_ACQUIRE_LOCK(base, th_base_lock);
	//如果有进位处理进位
	if (duration->tv_usec > 1000000) {
		memcpy(&tv, duration, sizeof(struct timeval));
		if (is_common_timeout(duration, base))
			tv.tv_usec &= MICROSECONDS_MASK;
		tv.tv_sec += tv.tv_usec / 1000000;
		tv.tv_usec %= 1000000;
		duration = &tv;
	}
	for (i = 0; i < base->n_common_timeouts; ++i) {
		const struct common_timeout_list *ctl =
		    base->common_timeout_queues[i];
		//设置的duration已经分配过common_timeout_list　 跳出
		if (duration->tv_sec == ctl->duration.tv_sec &&
		    duration->tv_usec ==
		    (ctl->duration.tv_usec & MICROSECONDS_MASK)) {
			EVUTIL_ASSERT(is_common_timeout(&ctl->duration, base));
			result = &ctl->duration;
			goto done;
		}
	}
	//超出限额 最多设置256个
	if (base->n_common_timeouts == MAX_COMMON_TIMEOUTS) {
		event_warnx("%s: Too many common timeouts already in use; "
		    "we only support %d per event_base", __func__,
		    MAX_COMMON_TIMEOUTS);
		goto done;
	}

	//如果空间不够，扩大数组队列 
	if (base->n_common_timeouts_allocated == base->n_common_timeouts) {
		int n = base->n_common_timeouts < 16 ? 16 :
		    base->n_common_timeouts*2;
		struct common_timeout_list **newqueues =
		    mm_realloc(base->common_timeout_queues,
			n*sizeof(struct common_timeout_queue *));
		if (!newqueues) {
			event_warn("%s: realloc",__func__);
			goto done;
		}
		base->n_common_timeouts_allocated = n;
		base->common_timeout_queues = newqueues;
	}
	//分配一个新的common_timeout_list,代表一个新的超时时间
	//作为list* 首元素
	new_ctl = mm_calloc(1, sizeof(struct common_timeout_list));
	if (!new_ctl) {
		event_warn("%s: calloc",__func__);
		goto done;
	}
	TAILQ_INIT(&new_ctl->events);
	new_ctl->duration.tv_sec = duration->tv_sec;
	new_ctl->duration.tv_usec =
	    duration->tv_usec | COMMON_TIMEOUT_MAGIC |
	    (base->n_common_timeouts << COMMON_TIMEOUT_IDX_SHIFT);
	//分配一个内部event, 注册回调为common_timeout_callback
	//类似signal里为socket pair的读端fd 注册了internal event的READ事件
	//evtime_assign表示仅监控的超时，不需要fd
	//注册回调为common_timeout_callback 参数为这个common_timeout结构
	evtimer_assign(&new_ctl->timeout_event, base,
	    common_timeout_callback, new_ctl);
	new_ctl->timeout_event.ev_flags |= EVLIST_INTERNAL;
	event_priority_set(&new_ctl->timeout_event, 0);
	new_ctl->base = base;
	base->common_timeout_queues[base->n_common_timeouts++] = new_ctl;
	result = &new_ctl->duration;

done:
	if (result)
		EVUTIL_ASSERT(is_common_timeout(result, base));

	EVBASE_RELEASE_LOCK(base, th_base_lock);
	return result;
}
```
### 回调 common_timeout_callback
在`common_timeout_callback`里,只激活队头的event,如果队列仍然有event，依据当前绝对时间重新event_add事件。

```cpp
/* Callback: invoked when the timeout for a common timeout queue triggers.
 * This means that (at least) the first event in that queue should be run,
 * and the timeout should be rescheduled if there are more events. */
static void
common_timeout_callback(evutil_socket_t fd, short what, void *arg)
{
	struct timeval now;
	struct common_timeout_list *ctl = arg;
	struct event_base *base = ctl->base;
	struct event *ev = NULL;
	EVBASE_ACQUIRE_LOCK(base, th_base_lock);
	gettime(base, &now);
	while (1) {
		//拿到队首event,应该是最近的发生的common_timeout事件。
		ev = TAILQ_FIRST(&ctl->events);
		//如果没有发生超时，则退出
		if (!ev || ev->ev_timeout.tv_sec > now.tv_sec ||
		    (ev->ev_timeout.tv_sec == now.tv_sec &&
			(ev->ev_timeout.tv_usec&MICROSECONDS_MASK) > now.tv_usec))
			break;
		//处理队列，第１个将remove掉，(下一次就是下一个)
		event_del_internal(ev);
		//将event激活，将执行回调
		event_active_nolock(ev, EV_TIMEOUT, 1);
	}
	//如果event不为空，用当前的绝对时间再event_add一次,即重新将会重新进入该回调
	if (ev)
		common_timeout_schedule(ctl, &now, ev);
	EVBASE_RELEASE_LOCK(base, th_base_lock);
}
```
### event_add中的common_timeout
event_add是在`event_add_internal`中，前面讲过了。
其中有这么一段代码，现在很清楚它在干嘛了:
```c
  gettime(base, &now);
   /*
    * 是否是common_timeout,是根据tv.usec的12bit来决定的
    * ４bit COMMON_TIMEOUT_MAGIC +  8bit index
    */
  common_timeout = is_common_timeout(tv, base);
  if (tv_is_absolute) {
    ev->ev_timeout = *tv;
  } else if (common_timeout) {
    //计算绝对时间
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

  //insert时，common_timeout将insert到自己的list
  event_queue_insert(base, ev, EVLIST_TIMEOUT);
  if (common_timeout) {
    struct common_timeout_list *ctl =
        get_common_timeout_list(base, &ev->ev_timeout);
    //如果是首元素，马上就会开始common_time的计时。
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

```
在`event_queue_insert`时:
```c 
case EVLIST_TIMEOUT: {
		//如果是common_timeout 放入common_timeout_list 而不是min heap
		if (is_common_timeout(&ev->ev_timeout, base)) {
			struct common_timeout_list *ctl =
			    get_common_timeout_list(base, &ev->ev_timeout);
			insert_common_timeout_inorder(ctl, ev);
		} else
			min_heap_push(&base->timeheap, ev);
		break;
	}
```
自然的，在`event_queue_remove`时，也需要处理:
```c 
case EVLIST_TIMEOUT:
  if (is_common_timeout(&ev->ev_timeout, base)) {
    struct common_timeout_list *ctl =
        get_common_timeout_list(base, &ev->ev_timeout);
    TAILQ_REMOVE(&ctl->events, ev,
        ev_timeout_pos.ev_next_with_common_timeout);
  } else {
    min_heap_erase(&base->timeheap, ev);
  }
  break;
```
