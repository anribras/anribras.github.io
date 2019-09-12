---
layout: post
title:
modified:
categories: Tech

tags: [web]

comments: true
---

<!-- TOC -->

- [基本概念](#基本概念)
  - [binding key and routing key](#binding-key-and-routing-key)
- [Worker 均衡](#Worker-均衡)
  - [场景举例](#场景举例)
- [发布订阅](#发布订阅)
- [远程调用](#远程调用)

<!-- /TOC -->

[Rabiitmq tutorials](http://www.rabbitmq.com/getstarted.html)

## 基本概念

`broker` RabbitMQ Server 就是 Message Broker,包含了 exchange 和 queue 的实体.

`producer` 生产者

`exchange` 发往 queue 的路由,不同 type 决定了不同的路由方式，有 fanout,direct,topic,headers

`queue`:队列 ,存储 producer 发来的消息的实体

`consumer`:消费者,连接到不同的 queue 消费消息.

`connect`是 producer or consumer 到 mq server 的 tcp 链接.

`channel`:一次连接的虚拟表示为 channel( exchange->queue) 1 个 producer 可能在 1 个 channel 建立多个 chanel.

### binding key and routing key

`binding key`是 chanel bind exchange 和 queue 时指定的,参数仍然叫 routing_key
其实就叫 bind routing key 好了
exchange 和 queuey 的 bind 关系，可以用这个 key 来表示.
甚至同样的 exchange 和 queue 可以 bind 到不同 key 名称，不过没啥意义.

```sh
exchage-A bind queue-B1 ,routing_key = 'ea.rb1'
exchage-A bind queue-B2 ,routing_key = 'ea.rb2'
```

到底有什么用?　后面 publish 的时候，可以指定具体走哪条 exchange-queue 的通道，就是通过 routing_key 来指示的.

真正起不起作用，又是 exchange 的 type 决定的，准确的说是`direct`,`topic`类型才有用.

```py
channel.exchange_declare(exchange=exchange_name, exchange_type='fanout')
channel.queue_bind(exchange=exchange_name,
                   queue=queue_name,
                   routing_key='black')
```

注意 queue_bind 是个很灵活的东西，可以:

- 1 exchanges <-> n queues
- n exchanges <-> 1 queues
- n exchanges <-> n queues
- 1 exchanges <-> 1 queues by by n routing-keys

栗子 1 个:

![Screenshot from 2018-11-08 10-42-30-1efba53d-d608-46d1-b1f8-b335d93356c7](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202018-11-08%2010-42-30-1efba53d-d608-46d1-b1f8-b335d93356c7.png)

另外在 publish 时，有可以指定 1 个 routing key,(这个就是前面设定的 bind routing_key)

```py
channel.basic_publish(exchange=exchange_name,
                        routing_key='black',
                        body=sent,
                        #make message persistent
                        properties=pika.BasicProperties(
                            delivery_mode=2,  # make message persistent
                        )
                        )
```

不同的 exchange type,对 binding key/routing key 有不同的用法:

> fanout 忽略此 key, 广播到所有连到 exchange 上的 queues
> direct 严格匹配, 消息仅发送到 routing key== binding key 的 queues
> topic 分组/模糊匹配, routing key = 'a.b.c', binding key = 'a.*.*c' or 'a.#', \*匹配 1 个单词，#匹配多个单词
> header 不再依赖 binding routing keys 而是消息中的 header 属性,

## Worker 均衡

默认 Round-Robin,即 1 queue 按顺序把 message 交给 worker(basic_consume 注册的对调)

如果设置了均衡(balance load):

```py
channel.basic_qos(prefetch_count=1)
```

即 server 发送 1 条消息，没收到回复时，(worker may busy),不会再给 queue 发送额外的消息，忙的继续忙，这就均衡了.

因此在 callback 里，一定记住:

```py
ch.basic_ack(delivery_tag = method.delivery_tag)
```

针对 1 个单独的 queue 的所有 wroker,才谈均衡.不管是 1 个 channel 的 n 个 worker，还是 n 个 channel 的 n 个 worker:

而多个 queue 的话，exchange 总是发送到匹配的多个 queue.

下面是针对 1 个 queue 的 4 个 worker,在 2 个 channel 里定义的:

```py
//start 4个worker in 2 channel
channel_1.basic_consume(callback,
                      queue="task_queue")

channel_1.basic_consume(callback,
                      queue="task_queue")

channel_2.basic_consume(callback,
                      queue="task_queue")

channel_2.basic_consume(callback,
                      queue="task_queue")
```

### 场景举例

某次访问，需要后台做出某种计算，根据访问的输入不同，计算的时长略有不同, 后台自然考虑是异步掉计算任务(hard task),
并且并行了 n 个集群 or 分布式的计算主体(task host)，任务来临时，通过类似 rabiitmq 把任务分发，要求计算时各主机是均衡的.
假设高峰时有 10000 次访问， 集群有 1000 个,每次任务 1s 左右.

## 发布订阅

exchange type 可以根据需求选择，但要点是: 1000 个 consumer 就要用 1000 个 queue, 即每个订阅者需要定义 1 个 queue,并将 queue bind 到 Publisher 的 exchange 上.

```py
#publisher
broadcast_exchange_name="ff.version.b"
channel.exchange_declare(exchange=broadcast_exchange_name,exchange_type='fanout')

#subscriber using anomynous queue name
result = channel.queue_declare(exclusive=True)
channel.queue_bind(queue=result.method.queue,
                   exchange=broadcast_exchange_name,
                   routing_key=broadcast_routing_key)
```

## 远程调用

远程调用需要把原输入序列化. 远程反序列化，调用结束后，再把成品(result)送回来

反序列化后还用到的技术有`反射`等.

从 client 端角度，可能都区分不清楚，RPC 是远调，还是 local call 了，远调可能更耗时一些.

异步角度来讲，少不了协程这些.

```py
some_rpc = RpcClient()
ret = some_prc.call('sth')
result = ret.get_result()
```

借用 Rabbitmq 来实现 RPC，原理同上,不过在 publish 时,指定好 callback_queue

```py

result = channel.queue_declare(exclusive=True)
callback_queue = result.method.queue

channel.basic_publish(exchange='',
                      routing_key='rpc_queue',
                      properties=pika.BasicProperties(
                            reply_to = callback_queue,
                            ),
                      body=request)
```

Properties 有 14 个，常用的 4 个是:

> delivery_mode = 2 消息持久化
> content_type = application/json 序列化方式
> rely_to = 指定回调通知的 queue
> corrlelation_id rpc 关联的 id ，每个 request 可以关联 1 个 unique id，respone 时好分辨

![Screenshot from 2018-11-08 11-04-00-c20fe086-b5e1-471d-a7ca-5aa86d869a06](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202018-11-08%2011-04-00-c20fe086-b5e1-471d-a7ca-5aa86d869a06.png)

rpc_server:

```py
import pika


def add_point(x,y):
    return int(x) + int(y)

class RpcServer:
    cb_queue_name = 'cbqueue'
    ','.split('.')

    def on_request(self,ch,method,properties,body):
        body = body.decode('utf-8')
        print('body = %s' % body)
        s = body.split('.')
        px,py = int(s[0]), int(s[1])
        ret = add_point(px,py)
        print('ret = %s' % ret)
        print('response to %s ',properties.reply_to)
        self.channel.basic_publish(
            exchange='',
            routing_key=properties.reply_to,
            properties=pika.BasicProperties(correlation_id = properties.correlation_id),
            body =str(ret)
        )


    def __init__(self):
        parameters = pika.ConnectionParameters(host='localhost', port=5672)
        self.connection = pika.BlockingConnection(parameters)
        self.channel = self.connection.channel()

        self.channel.queue_declare(queue='rpc_queue')

        self.channel.basic_consume(self.on_request,
                                   queue='rpc_queue'
                                   )



    def serve_forever(self):
        print('[RpcServer] Serve forever..')
        self.channel.start_consuming()


if __name__ == '__main__':
    RpcServer().serve_forever()

```

rpc_client:

```py
import pika
class Point():
    def __init__(self,x,y):
        self.x = x
        self.y = y

class RpcClient:

    def __init__(self):
        parameters = pika.ConnectionParameters(host='localhost', port=5672)
        self.connection = pika.BlockingConnection(parameters)
        self.channel = self.connection.channel()

        # self.channel.queue_declare(queue='rpc_queue',exclusive=True)

        result =  self.channel.queue_declare(exclusive=True)
        self.cb_queue_name = result.method.queue
        import  uuid
        self.cid = str(uuid.uuid4())
        self.result = None

        # basic comsume
        self.channel.basic_consume( self.on_response,
                               queue=self.cb_queue_name
        )

    def serialized(self,*args, **kwargs):
        # simple...
        # protobuf json sth in real
        return  str(args[0]) + '.' + str(args[1])

    def re_serialized(self,*args,**kwargs):
        return str(args[0])

    def on_response(self, ch, method, properties, body):
        # check id first
        if properties.correlation_id == self.cid:
            # reverse-serialied result
            print('on_response , body = %s' % body)
            self.result = self.re_serialized(body)

            self.channel.basic_ack(delivery_tag=method.delivery_tag)

    def call_add_point(self,px,py):
        self.result = None
        # serialized input:
        sent = self.serialized(px,py)


        # publish
        # default binding queue name as routing_key to exchange ''. default type fanout
        self.channel.basic_publish(exchange='',
                              routing_key='rpc_queue',
                              # make message persistent
                              properties=pika.BasicProperties(
                                  delivery_mode=2,  # make message persistent
                                  reply_to=self.cb_queue_name,
                                  correlation_id=self.cid
                              ),
                            body=sent
        )

        while self.result is None:
            # self.connection.process_data_events(time_limit=1)
            self.connection.process_data_events(time_limit=1)
            print('wait..')

        return self.result

if __name__ == '__main__':
    client = RpcClient()
    ret = client.call_add_point(1,2)
    print('ret = %s',ret)
```
