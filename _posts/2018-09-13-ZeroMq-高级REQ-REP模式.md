---
layout: post
title:
modified:
categories: Tech
 
tags: [web,zeromq]

  
comments: true
---
<!-- TOC -->

- [envelop机制](#envelop机制)
- [Combinations](#combinations)
- [The Load Balancing Pattern](#the-load-balancing-pattern)
    - [round robin  routing](#round-robin--routing)
    - [解决round robin 不足](#解决round-robin-不足)
- [The loading balance message broker](#the-loading-balance-message-broker)
- [High-level API](#high-level-api)

<!-- /TOC -->

## envelop机制

`The envelope is created by multiple sockets working together in a chain.`

所谓的envelop,由 address+delimiter+body构成，中间的delimter就是个空frame,用来分隔下数据.一个envelop可能是这样的:

```
addr1,addr2,.. + delimiter frame  + frame1 + frame2 +...
```

前面的address可以由发送端指定，也可以由系统默认生成.(5bytes:0+32bits random integers).

delimiter仅存在REQ发送的包里.

最简单的REQ-REP or (DEALER-REP) 通信,REQ socket会忽略address.REQ发送hello,,其实际数据是`delimiter +  data frame`的2个frame.而REP从收到的`envelop`剥离出data frame,然后再交给socket.recv.

但是从应用层上看，REP总是要recv one then send one .

而DEALER则灵活的多，async方式，可以send,send,send,recv,...send,recv..

对于ROUTER-DEALER，envelop里的address就有很用了，这也是实现routing的关键.

分析以下路径原理: `REQ<->ROUTER<->DEALER<->REP`

1. REQ->ROUTER

默认是`delimter + data frame`的`2 frames`;当然也可以在REQ socket里手动设置identity,直接变成`3 frames`;

2. ROUTER envelop 

如果么有adreess,ROUTER socket会把它接收的envelop(REQ->ROUTER)前面再加上1个address,总之是:`address + delimiter + data frame`的形式,即`3 frames`

ROUTER 内部应该是记录了1个类似hashmap的结构:addr:connection，ROUTER将当前标识为`addr`的envelop与具体的REQ connection对应

3. ROUTER->DEALER

ROUTER发送出去的必有address，DEALER拿到的是`3 frames`;

4. DEALER->REP

REP接收后,内部剥离`envelop`,(REP会保存envelop的结构,同样类似ROUTER的hashmap?)最终appliction拿到`data frame`.

5. REP  envelop

REP回复时，根据保存的envelop重新封装成`3 frames`，发送给DEALER;

6. DEALER->ROUTER

DEALER透传给ROUTER`3 frames`,DEALER不知道envelop机制的存在，仅把`3 frames`传输完就好.

7. ROUTER->REQ

ROUTER socket的内部会根据它要发送的envelop(ROUTER->DEALER)前面的address,也就是前面保存的hashmap,来决定如何做routing,并且发送给REQ时，已经去掉了address,即`delimiter + data frame`的`2 frames`. 

8. REQ processing

解envelop, 达到最终的`data frame`.

以某个ROUTER-DEALER的borker实现为例:
```py
#
#   Request-reply service in Python
#   Connects REP socket to tcp://localhost:5560
#   Expects "Hello" from client, replies with "World"
#
import zmq

# context = zmq.Context)
context = zmq.Context.instance()

frontend = context.socket(zmq.ROUTER)
frontend.bind("tcp://127.0.0.1:5559")
backend = context.socket(zmq.DEALER)
backend.bind("tcp://127.0.0.1:5560")
# Initialize poll set
poller = zmq.Poller()
poller.register(frontend, zmq.POLLIN)
poller.register(backend, zmq.POLLIN)

while True:
    socks = dict(poller.poll())

    if socks.get(frontend) == zmq.POLLIN:
        # why frontend.recv() not work! For the envelop reason
        message = frontend.recv_multipart()
        # [b'\x00k\x8bEg', b'', b'Hello 2']
        print('what msg in ROUTER recv? ', message)
        backend.send_multipart(message)

    if socks.get(backend) == zmq.POLLIN:
        message = backend.recv_multipart()
        print('what msg in DEALER recv? ', message)
        frontend.send_multipart(message)
```

可以看到,在ROUTER frontend接收时，不能仅用recv,而要recv_multipart,因为是1个envelop,具体的数据为:
```sh
# ROUTER
what msg in ROUTER recv?  [b'\x00k\x8bEg', b'', b'Hello 2']
# DEALER
what msg in DEALER recv?  [b'\x00k\x8bEg', b'', b'World']
# REP
Received request: [b'Hello 2']
$ REQ
Received reply 99 [b'World']
```
发送时，也是原封不动的将envelop交给DEALER.

## Combinations

合理的:

* REQ to REP  

REQ send and wait reply from REP ,REP wait then answwer to REQ,流程是严格固定的 

* DEALER to REP 

DEALER async方式，与任意多个connected的REP通信,不用等什么reply.

但是！发送给REP的数据，需要模拟成1个REQ的envelop的格式,否则REP是不认的，即附加1个`delimiter frame`

* REQ to ROUTER

就是proxy的一部分，转发给DEALER,前面描述过很多次.

或者用ROUTER做asynchronous server , 可以同时与多个REQ(client)通信.然是发送给REQ的仍然需要是envelop的形式，即`identity +  delimiter + data frame`.

* DEALER to ROUTER

DEALER(replace REQ) -> ROUTER -> DEALER-> ROUTER (replace REP).

消息随意传，每个结点都可以拿到所有传输的数据，无限制, async clients to async server.

* DEALER to DEALER

REQ->ROUTER->DEALER->DEALER(replace REP)

这种情况，末端的DEALER代替了REP,可以随意的reply,但是需要管理reply. 

* ROUTER to ROUTER

这个先不看..

不合理的:
```sh
REQ to REQ
REQ to DEALER
REP to REP
REP to ROUTER
```
理解了上面的envelop机制和各个角色的作用，不合理性应该好理解;

## The Load Balancing Pattern

ROUTE的高级用法.

### round robin  routing

首先理解下什么是`round robin routing`.

邮局n个窗口(queue)排队，客户来时，按窗口顺序让客户排队,就是round robin routing,比较简单.Zeromq里，就是用PUSH+DEALER实现了round bin routing.

如果每个客户处理时间一致，上述routing方法其实是很合理的.

实际上,有的客户慢，有的客户快，假设某个queue在一个慢客户处理期间，身后可能排了不少快客户在等待,对这些快客户其实是不公平的.

用1个`1 PUSH -> N DEALER`的例子:

```py
import zmq
import threading
import random
import time

context = zmq.Context()
DEALER_NUM = 5
sinks = []
for _ in range(DEALER_NUM):
    sinks.append(context.socket(zmq.DEALER))

for i in range(DEALER_NUM):
    sinks[i].connect("inproc://example")

def pull_worker(*args):
    idx = args[0]
    total = 0
    start = time.time()
    while True:
        msg = sinks[idx].recv_multipart()
        time.sleep( 1 * random.randint(1, 3))
        total += 1
        if total == 9:
            end = time.time()
            print('elapse time(%s) seconds for idx(%d) queue' % (end-start, idx))

threads = []
for _ in range(DEALER_NUM):
    threads.append( threading.Thread(target=pull_worker, args=[_]) )
for i in range(DEALER_NUM):
    threads[i].start()

anonymous = context.socket(zmq.PUSH)
anonymous.bind("inproc://example")

for i in range(DEALER_NUM * 10):
    anonymous.send(b"gogogo")

while True:
    time.sleep(1)

```
输出
```
elapse time(17.01690173149109) seconds for idx(2) queue
elapse time(17.019501447677612) seconds for idx(3) queue
elapse time(19.019424200057983) seconds for idx(0) queue
elapse time(20.018730640411377) seconds for idx(1) queue
elapse time(24.024351835250854) seconds for idx(4) queue
```
产生50个数据，通过PUSH分发到5个DEALER上，正是因为round robin策略，每个DEALER分配到了10个数据，即便每个数据的处理时间都是不一样的.
可以看到最快和最慢的queque差了有7秒，其实是有些浪费的.

### 解决round robin 不足

如果worker开始或完成1个任务时，能够通知调度官,调度管就可以把任务继续塞给它，就是这个思想.

为了完成来回的通知，worker选REQ(or DEALER),调度官则是选的ROUTER;只有ROUTER才能做到`谁发给我的，一会儿我再给你发回去`,因为envelop机制，可以先记住peer的addr,即下面调用的recv_mulitpart /send_mutilpart用的同一addr`

```py
#
# N REQ ->1 ROUTER to  demonstrate loading balance strategy
#
import zmq
import threading
import random
import time

context = zmq.Context()

PULL_NUM = 5

sinks = []
for _ in range(PULL_NUM):
    sinks.append(context.socket(zmq.REQ))

for i in range(PULL_NUM):
    sinks[i].connect("inproc://example")


def worker(*args):
    idx = args[0]
    total = 0
    start = time.time()
    while True:
        sinks[idx].send(b'ready')

        msg = sinks[idx].recv()
        if msg == b'end':
            end = time.time()
            print('elapse time(%s) seconds for idx(%d) queue, processed %d tasks' % (end - start, idx,total))
            break

        time.sleep(1 * random.randint(1, 3))
        # time.sleep(0.1)
        total += 1

threads = []
for _ in range(PULL_NUM):
    threads.append(threading.Thread(target=worker, args=[_]))
for i in range(PULL_NUM):
    threads[i].start()

anonymous = context.socket(zmq.ROUTER)
anonymous.bind("inproc://example")

for i in range(PULL_NUM * 10):
    [addr, empty, msg] = anonymous.recv_multipart()
    # print('addr %s msg %s' % (addr, msg))
    if msg == b'ready':
        # ROUTE send in `3 frames` format
        anonymous.send_multipart([addr, empty, b'gogogo'])

for i in range(PULL_NUM):
    [addr, empty, msg] = anonymous.recv_multipart()
    anonymous.send_multipart([addr, empty, b'end'])

```
输出:
```
elapse time(20.025699138641357) seconds for idx(1) queue, processed 9 tasks
elapse time(21.024115800857544) seconds for idx(3) queue, processed 10 tasks
elapse time(21.024389028549194) seconds for idx(4) queue, processed 9 tasks
elapse time(22.022366285324097) seconds for idx(2) queue, processed 11 tasks
elapse time(22.026769399642944) seconds for idx(0) queue, processed 11 tasks
```

可以看到同样不均衡的task，每个queue的使用时间几乎是均衡的.

## The loading balance message broker

之前讲过的模型:

![2018-09-11-10-09-47](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-11-10-09-47.png)

新模型很好的利用了上面的loading banlance.

![2018-09-14-14-26-50](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-09-14-14-26-50.png)

应用上仍然是client send and receive. 不过broker的处理经过很好的loading balance.

特别注意的是，broker是2个ROUTER组成的，而ROUTER对接收的消息会增加一层`envelop`,对发送的消息则会去掉一层`envelop`.

具体地:

1. 空闲worker先发出`ready`到backend ROUTER.
2. proxy检查哪个worker ready了(poll方式)，addr添加到 workerlist中，并开始监听client的请求
3. 有client的请求，则把workerlist中的队首的addr pop出来，讲client的具体消息发送到该work;
4. work进行处理;
5. work处理完毕,发送给backend ROUTER.
6. proxy 检查，区分消息是work返回的reply还是worker ready，前者的话，再给frontend ROUTER, frontend ROUTER交给对应的client;后者则添加到workerlist;
7. client 拿到reply

```py
# LRU queue using pyzmq IOLoop reactor
import threading
import time
from zmq.eventloop.ioloop import IOLoop
from zmq.eventloop.zmqstream import ZMQStream
import zmq
import random

NBR_CLIENTS = 50
NBR_WORKERS = 5

context = zmq.Context.instance()
localfd_url = "tcp://127.0.0.1:6000"
localbd_url = "tcp://127.0.0.1:6001"


def worker_task(ident):
    """
    REQ as worker
    """
    worker = context.socket(zmq.REQ)
    worker.identity = u"Worker-{}".format(ident).encode("ascii")
    worker.connect(localbd_url)

    # Tell broker , I'm ready
    worker.send(b'READY')

    count = 0

    total = 0

    # Processing tasks
    while True:
        try:
            start = time.time()
            [addr, empty, message] = worker.recv_multipart()
            time.sleep(random.randint(0, 3) * 0.1)
            # time.sleep(random.randint(0, 3) * 0.05)
            reply = u"Worker-{} process done".format(ident).encode("ascii")
            worker.send_multipart([addr, empty, reply])
            count += 1
            end = time.time()
            total += (end -start)
        except zmq.ContextTerminated:
            print('Worker-%d got %s tasks , spent %f seconds' % (ident, count, total))
            return

    pass


def client_task(ident):
    """
    REQ as client
    :param id:
    :return:
    """
    client = context.socket(zmq.REQ)
    client.connect(localfd_url)
    client.identity = u"Client-{}".format(ident).encode("ascii")
    # Send req and expect a rep , an normal REQ-REP flow
    client.send(b'Hello there, i am client-%d ' % ident)
    try:
        reply = client.recv()
        print('client-%d got reply:%s' % (ident, reply))
    except zmq.ContextTerminated:
        return


def single_broker_instance():
    localfd = context.socket(zmq.ROUTER)
    localbd = context.socket(zmq.ROUTER)

    localfd.bind(localfd_url)
    localbd.bind(localbd_url)

    # Add to poll
    poller = zmq.Poller()
    # Only poll for requests from backend until workers are available
    poller.register(localbd, zmq.POLLIN)

    workers = []
    work_ready = 0

    count = 0

    worked_started = 0

    while True:
        sockets = dict(poller.poll())

        # worker may got READY or worker reply
        if localbd in sockets:
            requests = localbd.recv_multipart()
            address, empty, messages = requests[:3]
            if not workers:
                poller.register(localfd, zmq.POLLIN)
            # Worker ready
            workers.append(address)
            if messages == b'READY' and len(requests) == 3:
                print('%s ready' % address)

                # Start clients then
                if worked_started == NBR_WORKERS - 1:
                    for i in range(NBR_CLIENTS):
                        threading.Thread(target=client_task, args=[i]).start()

                worked_started += 1

            # Messages sent from(ROUTER) to client be  [client address, empty, sth] , good for ROUTER routing.
            elif len(requests) == 5:
                client_address = messages
                empty, messages = requests[3:]
                localfd.send_multipart([client_address, empty, messages])
                count += 1
            else:
                print('some error')
                break
            # print(count)
            if count == NBR_CLIENTS - 1:
                print('quit')
                break

        # Got req from clients ROUTER->ROUTER
        if localfd in sockets:
            [address, empty, messages] = localfd.recv_multipart()
            print('valid workers:', workers)
            worker_address = workers.pop(0)
            localbd.send_multipart([worker_address, empty, address, empty, messages])
            if not workers:
                poller.unregister(localfd)
            pass

    localbd.close()
    localfd.close()
    context.term()
    pass

def broker_tasks(ident):
    pass


if __name__ == '__main__':


    for i in range(NBR_WORKERS):
        threading.Thread(target=worker_task, args=[i]).start()

    single_broker_instance()
```

输出:
```
Worker-0 got 11 tasks , spent 0.995400 seconds
Worker-1 got 9 tasks , spent 1.031592 seconds
Worker-3 got 7 tasks , spent 1.022296 seconds
Worker-2 got 9 tasks , spent 1.130031 seconds
Worker-4 got 13 tasks , spent 0.994066 seconds
```
可以看到每个worker拿到的task不一样(一样就是round robin了),但消耗的任务时间都差不多,这就是load balancing

## High-level API

是用更简练的API，完成上面的loading balance message borker.(http://zguide.zeromq.org/py:lbbroker3)

貌似简单不到哪里去...






