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
- [Worker均衡](#worker均衡)
    - [场景举例](#场景举例)
- [发布订阅](#发布订阅)
- [远程调用](#远程调用)

<!-- /TOC -->

[Rabiitmq tutorials](http://www.rabbitmq.com/getstarted.html)


## 基本概念

`producer` 生产者

`exchange` 发往queue的路由,不同type决定了不同的路由方式，有fanout,direct,topic,headers

`queue`:队列 

`consumer`:消费者

`channel`:一次连接的虚拟表示为channel 

### binding key and routing key

`binding key`是bind时指定的,参数仍然叫routing_key
也就是说，1个exchange和queue,可以指定不同的binding key
```py
channel.exchange_declare(exchange=exchange_name, exchange_type='fanout')
channel.queue_bind(exchange=exchange_name,
                   queue=queue_name,
                   routing_key='black')
```

注意queue_bind是个很灵活的东西，可以:
> 1 exchanges <-> n queues

> n exchanges <-> 1 queues

> n exchanges <-> n queues

> 1 exchanges <-> 1 queues by by n routing-keys

栗子1个:

![Screenshot from 2018-11-08 10-42-30-1efba53d-d608-46d1-b1f8-b335d93356c7](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202018-11-08%2010-42-30-1efba53d-d608-46d1-b1f8-b335d93356c7.png)


另外在publish时，有可以指定1个routing key:
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

不同的exchange type,对binding key/routing key有不同的用法:
> fanout忽略此key, 广播到所有连到exchange上的queues

> direct 严格匹配, 消息仅发送到routing key== binding key 的queues

> topic 分组/模糊匹配, routing key = 'a.b.c', binding key = 'a.*.*c' or 'a.#', *匹配1个单词，#匹配多个单词

> header 不再依赖binding routing keys 而是消息中的header属性, 



## Worker均衡

默认Round-Robin,即1 queue按顺序把message交给worker(basic_consume注册的对调)


如果设置了均衡(balance load):
```py
channel.basic_qos(prefetch_count=1)
```
即server发送1条消息，没收到回复时，(worker may busy),不会再给queue发送额外的消息，忙的继续忙，这就均衡了.

因此在callback里，一定记住:
```py
ch.basic_ack(delivery_tag = method.delivery_tag)
```

针对1个单独的queue的所有wroker,才谈均衡.不管是1个channel的n个worker，还是n个channel的n个worker:

而多个queue的话，exchange总是发送到匹配的多个queue.

下面是针对1个queue的4个worker,在2个channel里定义的:
```
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
并且并行了n个集群or分布式的计算主体(task host)，任务来临时，通过类似rabiitmq把任务分发，要求计算时各主机是均衡的. 
假设高峰时有10000次访问， 集群有1000个,每次任务1s左右.


## 发布订阅

exchange type可以根据需求选择，但要点是: 1000个consumer就要用1000个queue, 即每个订阅者需要定义1个queue,并将queue bind到Publisher的exchange上.
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

从client端角度，可能都区分不清楚，RPC是远调，还是local call了，远调可能更耗时一些. 

异步角度来讲，少不了协程这些.

```py
some_rpc = RpcClient()
ret = some_prc.call('sth')
result = ret.get_result()
```

借用Rabbitmq来实现RPC，原理同上,不过在publish时,指定好callback_queue
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

Properties有14个，常用的4个是:
> delivery_mode = 2 消息持久化

> content_type = application/json 序列化方式

> rely_to = 指定回调通知的queue

> corrlelation_id rpc关联的id ，每个request可以关联1个unique id，respone时好分辨

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


