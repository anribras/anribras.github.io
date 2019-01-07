---
layout: post
title:
modified:
categories: Tech
 
tags: [web,zeromq]

  
comments: true
---

<!-- TOC -->

- [base API](#base-api)
- [Multiple Sockets](#multiple-sockets)
- [Multipart Messages](#multipart-messages)
- [Unix Signal caughtion](#unix-signal-caughtion)
- [Multithreading with ZeroMQ](#multithreading-with-zeromq)
- [Signaling Between Threads](#signaling-between-threads)
- [Pub-sub Node Coordination](#pub-sub-node-coordination)
- [Pub-Sub Message Envelopes](#pub-sub-message-envelopes)
- [High-Water Marks](#high-water-marks)
- [Missing Message Problem Solver](#missing-message-problem-solver)

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

类似IO复用，zeromq也有poll机制，可方便同时读写多个socket.
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

消息组装，一个multipart message可以由多个frame来完成发送和接收.

发送时，最后一个message part发送完成时，整个multipatr message才开始发送;

接收时，第一个message part可以接收时，整个multipatr message实际上都已经接收了,这是很合理的.


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

用try-catch捕获,然后基于正常注册的signal hander该咋处理咋处理.
```c
    try {
        socket.recv (&msg);
    }
    catch(zmq::error_t& e) {
        std::cout << "W: interrupt received, proceeding…" << std::endl;
    }
```

或者判断返回的return code == EINTER,这个和unix保持一致.

## Multithreading with ZeroMQ

首先这里的multithreading 指的是server.

值得一提的是，Zeromq的多线程是`不使用任何同步原语`的高效实现.作者的比喻也是异常生动：`要真正的同步编程，就不要有任何共享的状态，否则就像2个醉汉抢1杯啤酒，他们必有一架干，酒桌上的啤酒越多，架就干的越多...`

![2018-09-12-16-01-45](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-12-16-01-45.png)

每个worker都是一个单线程，线程中`不要共享数据`.

DEALER和REP是inproc，即进程内的跨线程.

DEALER把task分配给REP(server),无需同步原语，由一定的策略决定.

## Signaling Between Threads 

线程间要同步通讯怎么办呢?就像`update->scale->rotate->convert->send`那样的pipeline线程流程?答案是`PARI sockets`.(个人怀疑这个能比同步原语快?)

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

node可以理解为一个网络节点，zeromq中的socket，proxy，都是node.如果node之间需要同步些消息(就像上面的threadh之间做的),用PAIR就不合适了，因为thread是一直运行在那的，而node在网络里，可能随时拿掉，又随时连上.

可以用`REQ-REP`,看一个Pub-sub 同步(sync pub-sub)的例子，pub需要知道多少sub连上了服务器，于是sub通过 req告知pub已连接，pub等预定数目的sub连接才真正的pub消息:

![2018-09-12-17-26-54](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-12-17-26-54.png)

## Pub-Sub Message Envelopes
其实就是传统pub-sub里的topic,subscriber可以仅订阅自己感兴趣的topic,Zeromq里用`envlope`.

```py
publisher.send_multipart([b"A", b"We don't want to see this"])
publisher.send_multipart([b"B", b"You want it"])
...
subscriber.setsockopt(zmq.SUBSCRIBE, b"B")
[address, contents] = subscriber.recv_multipart()
```

根据自己的需要可以扩充协议.

## High-Water Marks

在IO负载很高时，对socket的接收端或发送端(or both)的IO buffer行为进行控制，等待数据，丢弃接收数据等等.

## Missing Message Problem Solver

下面的流程处理也够详细了:

![](./img/Missing-Message-Problem-Solver.png)