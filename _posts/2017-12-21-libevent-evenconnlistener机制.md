---
layout: post
title:
modified:
categories: Tech
 
tags: [libevent,backend]

  
comments: true
---
<!-- TOC -->

- [前言](#前言)
- [evconnlistener_new](#evconnlistener_new)
- [evconnlistener_enable](#evconnlistener_enable)
- [evconnlistener_new_bind](#evconnlistener_new_bind)
- [listener_read_cb](#listener_read_cb)

<!-- /TOC -->

### 前言
libevent在对ready的fd绑定的event调用`event_assign,event_add`是其使用核心，但是fd的ready还是要经过socket的`socket,bind,listen,accept`,如何把这部分封装起来，就是eventconnlistener做的事情。
结构体的组织如下图:
![2017-12-21-11-36-20](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2017-12-21-11-36-20.png)

eventconnlistener里,有个简单的技巧，即通过结构体元素地址，寻找上层结构体的地址。
这样使用宏的技巧在libevent很多,如:

EVUTIL_UPCAST通过`lev`指针找到上层结构体`evconnlistener_event`的指针`lev_e`,其实目的就是为了使用`lev_e->listener` 即event。

```
struct evconnlistener_event *lev_e =
EVUTIL_UPCAST(lev, struct evconnlistener_event, base);


#define EVUTIL_UPCAST(ptr, type, field)				\
	((type *)(((char*)(ptr)) - evutil_offsetof(type, field)))

#define evutil_offsetof(type, field) offsetof(type, field)

#define offsetof(type, member) ((size_t) &((type *)0)->member)

```

### evconnlistener_new  

这个函数将`listen event_add event_assign`封装在了一起。

```c 
struct evconnlistener *
evconnlistener_new(struct event_base *base,
    evconnlistener_cb cb, void *ptr, unsigned flags, int backlog,
    evutil_socket_t fd)
{
	struct evconnlistener_event *lev;

	if (backlog > 0) {
		if (listen(fd, backlog) < 0)
			return NULL;
	} else if (backlog < 0) {
		if (listen(fd, 128) < 0)
			return NULL;
	}

	lev = mm_calloc(1, sizeof(struct evconnlistener_event));
	if (!lev)
		return NULL;

	lev->base.ops = &evconnlistener_event_ops;
	lev->base.cb = cb;
	lev->base.user_data = ptr;
	lev->base.flags = flags;
	lev->base.refcnt = 1;

	if (flags & LEV_OPT_THREADSAFE) {
		EVTHREAD_ALLOC_LOCK(lev->base.lock, EVTHREAD_LOCKTYPE_RECURSIVE);
	}

	event_assign(&lev->listener, base, fd, EV_READ|EV_PERSIST,
	    listener_read_cb, lev);

    //本质就是调用event_add
	evconnlistener_enable(&lev->base);

    //分配了evconnlistener_event,
    //也不必直接返回指针，而是成员结构体的指针，通过成员指针仍然找到自己。
	return &lev->base;
}


```
### evconnlistener_enable 
本质就是通过ops函数指针调用了`event_listener_enable`。
接着用到了上文提到的`EVUTIL_UPCAST`。兜兜转转，就是想拿到上层的结构体指针，然后对另外一个成员event使用`event_add`。
```
static int  
event_listener_enable(struct evconnlistener *lev)  
{  
    struct evconnlistener_event *lev_e =  
        EVUTIL_UPCAST(lev, struct evconnlistener_event, base);  
  
    //加入到event_base，完成监听工作。  
    return event_add(&lev_e->listener, NULL);  
}  
```
### evconnlistener_new_bind 
这个函数将`socket bind `以及上面的`evconnlistener_new `放在一起,
因此调用它就是完整的功能
```c 
struct evconnlistener *
evconnlistener_new_bind(struct event_base *base, evconnlistener_cb cb,
    void *ptr, unsigned flags, int backlog, const struct sockaddr *sa,
    int socklen)
{
	struct evconnlistener *listener;
	evutil_socket_t fd;
	int on = 1;
	int family = sa ? sa->sa_family : AF_UNSPEC;

	if (backlog == 0)
		return NULL;

	fd = socket(family, SOCK_STREAM, 0);
	if (fd == -1)
		return NULL;

	if (evutil_make_socket_nonblocking(fd) < 0) {
		evutil_closesocket(fd);
		return NULL;
	}

	if (flags & LEV_OPT_CLOSE_ON_EXEC) {
		if (evutil_make_socket_closeonexec(fd) < 0) {
			evutil_closesocket(fd);
			return NULL;
		}
	}

	if (setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, (void*)&on, sizeof(on))<0) {
		evutil_closesocket(fd);
		return NULL;
	}
	if (flags & LEV_OPT_REUSEABLE) {
		if (evutil_make_listen_socket_reuseable(fd) < 0) {
			evutil_closesocket(fd);
			return NULL;
		}
	}

	if (sa) {
		if (bind(fd, sa, socklen)<0) {
			evutil_closesocket(fd);
			return NULL;
		}
	}

	listener = evconnlistener_new(base, cb, ptr, flags, backlog, fd);
	if (!listener) {
		evutil_closesocket(fd);
		return NULL;
	}

	return listener;
}

```

### listener_read_cb
是在`event_assign`里指定了listen到了fd后的回调。
```
event_assign(&lev->listener, base, fd, EV_READ|EV_PERSIST,                       listener_read_cb, lev);
```
accept的调用自然是在这个回调里。在回调里自然也实现了accept后的回调`evconnlistener_cb`。
```
static void
listener_read_cb(evutil_socket_t fd, short what, void *p)
{
	struct evconnlistener *lev = p;
	int err;
	evconnlistener_cb cb;
	evconnlistener_errorcb errorcb;
	void *user_data;
	LOCK(lev);
	while (1) {
		struct sockaddr_storage ss;
#ifdef WIN32
		int socklen = sizeof(ss);
#else
		socklen_t socklen = sizeof(ss);
#endif
		evutil_socket_t new_fd = accept(fd, (struct sockaddr*)&ss, &socklen);
		if (new_fd < 0)
			break;
		if (socklen == 0) {
			/* This can happen with some older linux kernels in
			 * response to nmap. */
			evutil_closesocket(new_fd);
			continue;
		}

		if (!(lev->flags & LEV_OPT_LEAVE_SOCKETS_BLOCKING))
			evutil_make_socket_nonblocking(new_fd);

		if (lev->cb == NULL) {
			evutil_closesocket(new_fd);
			UNLOCK(lev);
			return;
		}
		++lev->refcnt;
		cb = lev->cb;
		user_data = lev->user_data;
		UNLOCK(lev);
        //用户注册的evconnlisten_cb在这里被调用
		cb(lev, new_fd, (struct sockaddr*)&ss, (int)socklen,
		    user_data);
		LOCK(lev);
		if (lev->refcnt == 1) {
			int freed = listener_decref_and_unlock(lev);
			EVUTIL_ASSERT(freed);
			return;
		}
		--lev->refcnt;
	}
	err = evutil_socket_geterror(fd);
	if (EVUTIL_ERR_ACCEPT_RETRIABLE(err)) {
		UNLOCK(lev);
		return;
	}
	if (lev->errorcb != NULL) {
		++lev->refcnt;
		errorcb = lev->errorcb;
		user_data = lev->user_data;
		UNLOCK(lev);
		errorcb(lev, user_data);
		LOCK(lev);
		listener_decref_and_unlock(lev);
	} else {
		event_sock_warn(fd, "Error from accept() call");
	}
}
```

