---
layout: post
title:
modified:
categories: Tech
 
tags: [web,zeromq]

  
comments: true
---

<!-- TOC -->

- [socket概念](#socket概念)
- [message](#message)
    - [IO Thread](#io-thread)
    - [4 message patterns](#4-message-patterns)
        - [Req-Rep](#req-rep)
        - [Publish-Subscribe](#publish-subscribe)
        - [Divide and Conquer(Pull and Push)](#divide-and-conquerpull-and-push)
        - [Excluesive pair](#excluesive-pair)
        - [Sumerize](#sumerize)

<!-- /TOC -->


## socket概念

```c
zmq_socket()
zmq_close()

zmq_setsockopt()
zmq_getsockopt()

zmq_bind()
zmq_connect()
```

zero mq的socket不同与raw socket的socket,可理解为socket的本质是一个`实现了异步的消息队列抽象实体`,它的特点:

> socket底下可以维持多个ougoing or incoming connections.

> no zmq_accpet() method

> client 可以先与server建立;

`Clients and servers can connect and bind at any time, can go and come back, and it remains transparent to applications.`

尽管模糊了server与client,但是server仍然是固定addr的一方，client仍然是动态变化的一方.

> 基本网络类型有: inproc, ipc,tcp,pgm,egpm

socket类型决定了它是怎么routing,queueing messages的.

对于inporc, server 必须先bind.

> a socket可以bind多个不用的网络类型,但是同一类型的port不能绑定多次，只有ipc是例外.

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

socket的传输的基本类型是message frame,不再是raw binary.

zeromq 可以理解在tcp上的封装,和http一个级别，再封装下可以实现任何协议.

[具体的ZMTP内容](https://rfc.zeromq.org/spec:15/ZMTP/)


### IO Thread 

messages在收发两端的buffer io都是做好的. 每个socket有一个IO Thread处理IO.,就是最开始的zmq_ctx_new()创造出来的.比如send,不是send结束就发送完毕了，可能正在io缓冲中.

定义多个IO thread:
```c
int io_threads = 4;
void *context = zmq_ctx_new ();
zmq_ctx_set (context, ZMQ_IO_THREADS, io_threads);
assert (zmq_ctx_get (context, ZMQ_IO_THREADS) == io_threads);
```
inproc(多线程)应用下，设置为0都可以，因为没有外部IO需求.

### 4 message patterns

#### Req-Rep

`REQ-REP`就是简单的cliend send and recv, server recv and send模型.

![2018-09-05-10-30-20](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-05-10-30-20.png)

这种模式下有4个角色:`ZMQ_REQ`,`ZMQ_REP`,`ZMQ_DEALER`,`ZMQ_ROUTER`

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

就是很传统的pub-sub实现.需要注意的1点，如果先sub,后pub,那么subscribe拿到的第1条pub信息`一定会丢失`.因为sub的第1次接收需要处理连接，可能not ready的时候，message已经发出去了.

一共有`ZMQ_PUB,ZMQ_XPUB,ZMQ_SUB,ZMQ_XSUB`4种socket types.

ZMQ_PUB:

![2018-09-11-14-58-24](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-14-58-24.png)

ZMQ_SUB:

![2018-09-11-14-58-38](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-14-58-38.png)

传统的PUB->SUB是单向的，而XPUB<->XSUB允许双向通知,XPUB+XSUB用来实现pub-sub的中间proxy,后面会讲到.

ZMQ_XPUB:

![2018-09-11-14-59-05](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-14-59-05.png)

ZMQ_XSUB:

![2018-09-11-14-59-22](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-14-59-22.png)


#### Divide and Conquer(Pull and Push)

也叫Pipeline模式.

![2018-09-05-11-37-57](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-05-11-37-57.png)

单理解为push and pull有点曲解它的意思了，应该是分而治之.ventilator生产任务，worker可并行消费，最后sink汇总. worker 并行时，可能会有`load balance`问题，不同的worker处理速度不一致，导致性能不佳, 

有`ZMQ_PUSH,ZMQ_PULL`2个types:

ZMQ_PUSH:

![2018-09-11-15-08-37](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-15-08-37.png)

ZMQ_PULL:

![2018-09-11-15-08-46](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-15-08-46.png)

#### Excluesive pair

可以理解为点对点，双向任意通信，就像socket pair一样,只有1个`ZMQ_PAIR`角色:

![2018-09-11-15-10-17](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-15-10-17.png)

#### Sumerize

上面表格里列出的mute state，指接收时buffer到达high water mark或者发送时低于low water maker(一般就是0啦)时，对数据的处理行为:`Block` or `Drop`

incoming routing strategy:`last peer` or `faired queue`


outgoing routing strategy:`round robin`

