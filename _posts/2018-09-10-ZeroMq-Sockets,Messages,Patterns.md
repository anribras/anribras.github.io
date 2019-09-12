---
layout: post
title:
modified:
categories: Tech

tags: [web, zeromq]

comments: true
---

<!-- TOC -->

- [socket 概念](#socket-概念)
- [message](#message)
  - [IO Thread](#IO-Thread)
  - [4 message patterns](#4-message-patterns)
    - [Req-Rep](#Req-Rep)
    - [Publish-Subscribe](#Publish-Subscribe)
    - [Divide and Conquer(Pull and Push)](#Divide-and-ConquerPull-and-Push)
    - [Excluesive pair](#Excluesive-pair)
    - [Sumerize](#Sumerize)

<!-- /TOC -->

## socket 概念

```c
zmq_socket()
zmq_close()

zmq_setsockopt()
zmq_getsockopt()

zmq_bind()
zmq_connect()
```

zero mq 的 socket 不同与 raw socket 的 socket,可理解为 socket 的本质是一个`实现了异步的消息队列抽象实体`,它的特点:

> socket 底下可以维持多个 ougoing or incoming connections.
>
> no zmq_accpet() method
>
> client 可以先与 server 建立;

`Clients and servers can connect and bind at any time, can go and come back, and it remains transparent to applications.`

尽管模糊了 server 与 client,但是 server 仍然是固定 addr 的一方，client 仍然是动态变化的一方.

> 基本网络类型有: inproc, ipc,tcp,pgm,egpm

socket 类型决定了它是怎么 routing,queueing messages 的.

对于 inporc, server 必须先 bind.

> a socket 可以 bind 多个不用的网络类型,但是同一类型的 port 不能绑定多次，只有 ipc 是例外.

```c
zmq_bind (socket, "tcp://*:5555");
zmq_bind (socket, "tcp://*:9999");
zmq_bind (socket, "inproc://somename");

zmq_bind (socket, "ipc://s1");
//表示如果前面的s1挂掉，后面的s1作为备份.
zmq_bind (socket, "ipc://s1");
// bind后也可以connect
zmq_connect (socket, "ipc://s2");
```

## message

socket 的传输的基本类型是 message frame,不再是 raw binary.

zeromq 可以理解在 tcp 上的封装,和 http 一个级别，再封装下可以实现任何协议.

[具体的 ZMTP 内容](https://rfc.zeromq.org/spec:15/ZMTP/)

### IO Thread

messages 在收发两端的 buffer io 都是做好的. 每个 socket 有一个 IO Thread 处理 IO.,就是最开始的 zmq_ctx_new()创造出来的.比如 send,不是 send 结束就发送完毕了，可能正在 io 缓冲中.

定义多个 IO thread:

```c
int io_threads = 4;
void *context = zmq_ctx_new ();
zmq_ctx_set (context, ZMQ_IO_THREADS, io_threads);
assert (zmq_ctx_get (context, ZMQ_IO_THREADS) == io_threads);
```

inproc(多线程)应用下，设置为 0 都可以，因为没有外部 IO 需求.

### 4 message patterns

#### Req-Rep

`REQ-REP`就是简单的 cliend send and recv, server recv and send 模型.

![2018-09-05-10-30-20](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-05-10-30-20.png)

这种模式下有 4 个角色:`ZMQ_REQ`,`ZMQ_REP`,`ZMQ_DEALER`,`ZMQ_ROUTER`

REQ:

![2018-09-11-09-54-08](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-09-54-08.png)

REP:

![2018-09-11-09-54-23](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-09-54-23.png)

DEALER:

![2018-09-11-09-54-39](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-09-54-39.png)

ROUTER:

![2018-09-11-09-54-52](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-09-54-52.png)

详细理解,参考后面的`高级REQ-REP模式`.

#### Publish-Subscribe

![2018-09-05-10-34-18](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-05-10-34-18.png)

就是很传统的 pub-sub 实现.需要注意的 1 点，如果先 sub,后 pub,那么 subscribe 拿到的第 1 条 pub 信息`一定会丢失`.因为 sub 的第 1 次接收需要处理连接，可能 not ready 的时候，message 已经发出去了.

一共有`ZMQ_PUB,ZMQ_XPUB,ZMQ_SUB,ZMQ_XSUB`4 种 socket types.

ZMQ_PUB:

![2018-09-11-14-58-24](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-14-58-24.png)

ZMQ_SUB:

![2018-09-11-14-58-38](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-14-58-38.png)

传统的 PUB->SUB 是单向的，而 XPUB<->XSUB 允许双向通知,XPUB+XSUB 用来实现 pub-sub 的中间 proxy,后面会讲到.

ZMQ_XPUB:

![2018-09-11-14-59-05](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-14-59-05.png)

ZMQ_XSUB:

![2018-09-11-14-59-22](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-14-59-22.png)

#### Divide and Conquer(Pull and Push)

也叫 Pipeline 模式.

![2018-09-05-11-37-57](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-05-11-37-57.png)

单理解为 push and pull 有点曲解它的意思了，应该是分而治之.ventilator 生产任务，worker 可并行消费，最后 sink 汇总. worker 并行时，可能会有`load balance`问题，不同的 worker 处理速度不一致，导致性能不佳,

有`ZMQ_PUSH,ZMQ_PULL`2 个 types:

ZMQ_PUSH:

![2018-09-11-15-08-37](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-15-08-37.png)

ZMQ_PULL:

![2018-09-11-15-08-46](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-15-08-46.png)

#### Excluesive pair

可以理解为点对点，双向任意通信，就像 socket pair 一样,只有 1 个`ZMQ_PAIR`角色:

![2018-09-11-15-10-17](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-15-10-17.png)

#### Sumerize

上面表格里列出的 mute state，指接收时 buffer 到达 high water mark 或者发送时低于 low water maker(一般就是 0 啦)时，对数据的处理行为:`Block` or `Drop`

incoming routing strategy:`last peer` or `faired queue`

outgoing routing strategy:`round robin`
