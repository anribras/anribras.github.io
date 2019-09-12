---
layout: post
title:
modified:
categories: Tech

tags: [web, zeromq]

comments: true
---

<!-- TOC -->

- [base API](#base-API)
- [Multiple Sockets](#Multiple-Sockets)
- [Multipart Messages](#Multipart-Messages)
- [Unix Signal caughtion](#Unix-Signal-caughtion)
- [Multithreading with ZeroMQ](#Multithreading-with-ZeroMQ)
- [Signaling Between Threads](#Signaling-Between-Threads)
- [Pub-sub Node Coordination](#Pub-sub-Node-Coordination)
- [Pub-Sub Message Envelopes](#Pub-Sub-Message-Envelopes)
- [High-Water Marks](#High-Water-Marks)
- [Missing Message Problem Solver](#Missing-Message-Problem-Solver)

<!-- /TOC -->

## base API

Initialise a message:

```c
zmq_msg_init()
zmq_msg_init_size()
zmq_msg_init_data().
```

Sending and receiving a message:

```c
zmq_msg_send()
zmq_msg_recv()
```

Release a message:

```c
zmq_msg_close().
```

Access message content:

```c
zmq_msg_data()
zmq_msg_size()
zmq_msg_more()
```

Work with message properties

```c
zmq_msg_get()
zmq_msg_set()
```

Message manipulation

```c
zmq_msg_copy()
zmq_msg_move()
```

## Multiple Sockets

类似 IO 复用，zeromq 也有 poll 机制，可方便同时读写多个 socket.

```c
//
//  Reading from multiple sockets in C++
//  This version uses zmq_poll()
//
// Olivier Chamoux <olivier.chamoux@fr.thalesgroup.com>

#include "zhelpers.hpp"

int main (int argc, char *argv[])
{
    zmq::context_t context(1);

    //  Connect to task ventilator
    zmq::socket_t receiver(context, ZMQ_PULL);
    receiver.connect("tcp://localhost:5557");

    //  Connect to weather server
    zmq::socket_t subscriber(context, ZMQ_SUB);
    subscriber.connect("tcp://localhost:5556");
    subscriber.setsockopt(ZMQ_SUBSCRIBE, "10001 ", 6);

    //  Initialize poll set
    zmq::pollitem_t items [] = {
        { receiver, 0, ZMQ_POLLIN, 0 },
        { subscriber, 0, ZMQ_POLLIN, 0 }
    };
    //  Process messages from both sockets
    while (1) {
        zmq::message_t message;
        zmq::poll (&items [0], 2, -1);

        if (items [0].revents & ZMQ_POLLIN) {
            receiver.recv(&message);
            //  Process task
        }
        if (items [1].revents & ZMQ_POLLIN) {
            subscriber.recv(&message);
            //  Process weather update
        }
    }
    return 0;
}
```

## Multipart Messages

消息组装，一个 multipart message 可以由多个 frame 来完成发送和接收.

发送时，最后一个 message part 发送完成时，整个 multipatr message 才开始发送;

接收时，第一个 message part 可以接收时，整个 multipatr message 实际上都已经接收了,这是很合理的.

Send:

```c
zmq_msg_send (&message, socket, ZMQ_SNDMORE);
…
zmq_msg_send (&message, socket, ZMQ_SNDMORE);
…
zmq_msg_send (&message, socket, 0);
```

Receive:

```c
while (1) {
    zmq_msg_t message;
    zmq_msg_init (&message);
    zmq_msg_recv (&message, socket, 0);
    //  Process the message frame
    …
    zmq_msg_close (&message);
    if (!zmq_msg_more (&message))
        break;      //  Last message frame
}
```

## Unix Signal caughtion

用 try-catch 捕获,然后基于正常注册的 signal hander 该咋处理咋处理.

```c
    try {
        socket.recv (&msg);
    }
    catch(zmq::error_t& e) {
        std::cout << "W: interrupt received, proceeding…" << std::endl;
    }
```

或者判断返回的 return code == EINTER,这个和 unix 保持一致.

## Multithreading with ZeroMQ

首先这里的 multithreading 指的是 server.

值得一提的是，Zeromq 的多线程是`不使用任何同步原语`的高效实现.作者的比喻也是异常生动：`要真正的同步编程，就不要有任何共享的状态，否则就像2个醉汉抢1杯啤酒，他们必有一架干，酒桌上的啤酒越多，架就干的越多...`

![2018-09-12-16-01-45](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-12-16-01-45.png)

每个 worker 都是一个单线程，线程中`不要共享数据`.

DEALER 和 REP 是 inproc，即进程内的跨线程.

DEALER 把 task 分配给 REP(server),无需同步原语，由一定的策略决定.

## Signaling Between Threads

线程间要同步通讯怎么办呢?就像`update->scale->rotate->convert->send`那样的 pipeline 线程流程?答案是`PARI sockets`.(个人怀疑这个能比同步原语快?)

```python
import threading
import zmq
import time

context = zmq.Context()

def step1(contexts=None):
    ctx = contexts or zmq.Context.instance()
    sender = context.socket(zmq.PAIR)
    sender.bind("inproc://b1")
    time.sleep(1)
    sender.send(b'origin stuff')
    pass

def step2(contexts=None):

    receiver = context.socket(zmq.PAIR)
    receiver.connect("inproc://b1")
    msg = receiver.recv_multipart()
    print("Got from b1, ", msg)

    time.sleep(1)

    sender = context.socket(zmq.PAIR)
    sender.bind("inproc://b2")
    sender.send(b'scaling')
    pass

def step3(contexts=None):

    receiver = context.socket(zmq.PAIR)
    receiver.connect("inproc://b2")
    msg = receiver.recv_multipart()
    print("Got from b2, ", msg)

    time.sleep(1)

    sender = context.socket(zmq.PAIR)
    sender.bind("inproc://b3")
    sender.send(b'converting')
    pass

def step4(contexts=None):

    ctx = contexts or zmq.Context.instance()

    receiver = context.socket(zmq.PAIR)
    receiver.connect("inproc://b3")
    msg = receiver.recv_multipart()
    print("Got from b3, ", msg)

    time.sleep(1)
    print('things done')
    pass

if __name__ == '__main__':
    t1 = threading.Thread(target=step1, args=[context])
    t2 = threading.Thread(target=step2, args=[context])
    t3 = threading.Thread(target=step3, args=[context])
    t4 = threading.Thread(target=step4, args=[context])

    t1.start()
    t2.start()
    t3.start()
    t4.start()

    t1.join()
    t2.join()
    t3.join()
    t4.join()
    pass
```

## Pub-sub Node Coordination

node 可以理解为一个网络节点，zeromq 中的 socket，proxy，都是 node.如果 node 之间需要同步些消息(就像上面的 threadh 之间做的),用 PAIR 就不合适了，因为 thread 是一直运行在那的，而 node 在网络里，可能随时拿掉，又随时连上.

可以用`REQ-REP`,看一个 Pub-sub 同步(sync pub-sub)的例子，pub 需要知道多少 sub 连上了服务器，于是 sub 通过 req 告知 pub 已连接，pub 等预定数目的 sub 连接才真正的 pub 消息:

![2018-09-12-17-26-54](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-12-17-26-54.png)

## Pub-Sub Message Envelopes

其实就是传统 pub-sub 里的 topic,subscriber 可以仅订阅自己感兴趣的 topic,Zeromq 里用`envlope`.

```py
publisher.send_multipart([b"A", b"We don't want to see this"])
publisher.send_multipart([b"B", b"You want it"])
...
subscriber.setsockopt(zmq.SUBSCRIBE, b"B")
[address, contents] = subscriber.recv_multipart()
```

根据自己的需要可以扩充协议.

## High-Water Marks

在 IO 负载很高时，对 socket 的接收端或发送端(or both)的 IO buffer 行为进行控制，等待数据，丢弃接收数据等等.

## Missing Message Problem Solver

下面的流程处理也够详细了:

![sovler](./img/Missing-Message-Problem-Solver.png)
