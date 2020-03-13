---
layout: post
title:
modified:
categories: Tech

tags: [web]

comments: true
---

<!-- TOC -->

- [设置](#设置)
- [原理](#原理)

<!-- /TOC -->

## 设置

[nginx 增强理解](https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching)

nginx 增加下面的配置:

```sh
    location /ver {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        #default Nginx set http header as itself as hostname
        proxy_set_header Host $proxy_host;
        proxy_connect_timeout 5s;
        proxy_send_timeout 15s;
        proxy_read_timeout 86400s; //默认60s nginx会backend断开ws，那肯定不行，设为1天
        proxy_pass http://localhost:9010;
    }
```

其中`http://localhost:9010`是 ws 的 listening server.

当 client 访问`ws://server_name/ver`时，nginx 将反向到 ws server 上.

特别注意`proxy_set_header Host $proxy_host;`的设置,之前不行就是因为这个地方没对.

nginx 默认就是这个设置，这里重新设置下，是因为 nginx server 里设置了`proxy_set_header Host $host;`

proxy_set_header 看这里<https://www.jianshu.com/p/0850db5af284>

anyway, 是设置 nginx 给转发目的的 http header 的`HOST`字段

```sh

//设置HOST字段==原始http req header里的HOST
proxy_set_header Host $http_host;
//同上，但是如果header没有这个字段，则设置为nginx里的server_name(理论上就是你访问的url，如www.163.com)
//一般用这个就好
proxy_set_header Host $host;


//设置HOST字段为proxy_pass 后跟着的HOST
proxy_set_header Host $proxy_host;
//下面是proxy_host = 127.0.0.1
1. proxy_pass http://localhost:9010

假如backends对应的upstream里的server www.xxx.net的ip是10.1.1.1,那么proxy_host就是这个值
2. proxy_pass http://backends

upstream  backends {
    server www.xxx.net
}
```

顺便说说 upstream,这才是反向代理的精髓，比如大家都访问`www.xxx.net/res/`,通过 upstream，把访问均衡到多个 web server 上. upstream 可以选择很多`均衡策略`.

```sh
server {
    listen *:80 default_server;
    server_name www.xxx.net;
    location /res {
        proxy_pass http://backends;
    }
}

upstream backends {
    server http://localhost:8080;
    server http://localhost:8081;
    server http://localhost:8082;
    server 10.20.1.2:8080;
    server 10.20.1.2:8081;
    ...

}
```

## 原理

要是知道 websocket 的握手是借用了 http 的,一开始的 request header:

```sh
 --- request header ---
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Host: 127.0.0.1:8001
Origin: http://127.0.0.1:8001
Sec-WebSocket-Key: hj0eNqbhE/A0GkBXDRrYYw==
Sec-WebSocket-Version: 13
```

下面是 respone 字段:

```sh
  HTTP/1.1 101 Switching Protocols
Content-Length: 0
Upgrade: websocket
Sec-Websocket-Accept: ZEs+c+VBk8Aj01+wJGN7Y15796g=
Server: TornadoServer/4.5.1
Connection: Upgrade
Date: Wed, 21 Jun 2017 03:29:14 GMT
```

验证过程:

1. client 发送随机 key`Sec-WebSocket-Key`给 server,
2. server 进行 sha1 哈希(+魔数),返回`Sec-Websocket-Accept`

```py
    def compute_accept_value(key):
        """Computes the value for the Sec-WebSocket-Accept header,
        given the value for Sec-WebSocket-Key.
        """
        sha1 = hashlib.sha1()
        sha1.update(utf8(key))
        sha1.update(b"258EAFA5-E914-47DA-95CA-C5AB0DC85B11")  # Magic value
        return native_str(base64.b64encode(sha1.digest()))
```

client 进行同样的操作，验证了 server,握手完成
