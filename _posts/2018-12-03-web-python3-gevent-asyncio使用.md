---
layout: post
title:
modified:
categories: Tech
 
tags: [python]

comments: true
---


<!-- TOC -->

- [协程](#协程)
- [gevent](#gevent)
- [asyncio](#asyncio)

<!-- /TOC -->


## 协程


从单进程模型开始,后续三条思路:
```
多进程/多线程,每条处理实际是同步阻塞的,不过借用OS调度到不同的执行体上
reactor/proactor
协程coroutinue
```

多进程/多线程遇到的瓶颈是`c10k`问题.即单机处理并发用多进程这种方式.10k就是瓶颈,增加集群就是成本.

reactor/proactor是异步非阻塞,eventloop,很好很强大,但是回调多了后,也造成了`callback hell`,业务逻辑被各种callback打的稀碎.

协程出现的很早,但是早期并没有像reactor那样大放异彩.协程使用时,看上去像同步阻塞,实际是协程通过主动让出(yield)让空闲的协程执行完毕后再(resume).而且整个调度是在应用层(epoll)完成,不像os调度那样频繁切换上下文.

其通俗理解：让原来要使用异步+回调方式(reactor or proactor)写的非人类代码,可以用看似同步的方式写出来. 


深入再看看:

[协程的好处](https://www.zhihu.com/question/20511233)

## gevent

gevent的前身是greenlet,是用协程实现的并行框架,和它一个级别的兄弟还有:Twisted,libevent,libuv等等.

gevent是单进程模型

monkey_patch给python封装的socket,select等模块打补丁才能支持gevent;这个patch可能带来问题, 使用SSL sockets, WSGI handler, gunicorn, AMQP这些时要特别注意.

比python multiprocessing/threading如何?

```
Python runtime bound: use multiprocessing
IO bound: use gevent
Don't want to fight gevent gotchas: use threading.
```
[multiprocessing_vs_threads_vs_gevents](https://www.reddit.com/r/Python/comments/3v5i0a/multiprocessing_vs_threads_vs_gevents/)


## asyncio

asyncio python自带的协程实现,关键字支持 await async函数定义等等.
