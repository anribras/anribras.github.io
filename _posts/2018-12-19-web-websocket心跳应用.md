---
layout: post
title:
modified:
categories: Tech

tags: [python]

comments: true
---

<!-- TOC -->

- [User story](#User-story)
- [不足分析](#不足分析)
- [逻辑实现](#逻辑实现)
- [新的问题](#新的问题)

<!-- /TOC -->

websocket 稳定 push 设计方案

## User story

1000 台客户端同时在线, 突然发现 bug, 导致大面积瘫痪, server 使用 websocket 做更新推送.

版本紧急修复后,上线实现步骤:

1. 修复版本
2. admin 登录 portal
3. 上传当前补丁信息, 设置发布条件: push 时间, 车机 id, 地理限制, 人数限制, push 频率...等等.
4. 设置完成后, 页面显示 push 目标人数, 即当前补丁正式 push 前的在线人数.
5. push 操作,点击后开始执行 push, 后台按设定策略开始通知在线车机.

当前实现,可以解决在 push 期间出现 client 异常掉线/新用户上线的场景.

1. 用 python list 存储 [ client object1 ,... ],client 增加属性 flag=0
2. push 时,对每个 client 发送命令. 发送完毕的 client flag=1.
    循环检测 list 里 dict 元素的 flag,直到最后 1 个 flag 不为 0,即所有 client 被 push 了,才 push end.
    push 期间, 再次 push 时 trylock
    push 期间 ,重连上线或者新上线的 client 可以处理(client 会 append 到 list)
    push 期间 ,如果 client 掉线, remove(client).

## 不足分析

上线步骤 4 和 5 期间, 考虑可能发生在以下任意时间段(t0,t1,t2,t3)的掉线/重新上线:

---t0 ----- tinker ready --- t1 ---push start ---t2--- push end -----t3------

t0 之后的新用户 由 app 自己处理;
在 t2 之前因为掉线而重新上线的,都由上面已经实现的机制处理;
即主要考虑在 t0/t1/t2 掉线,但在 t3 又重连的客户,如何识别?

t3 连上的客户可能包括已下 3 种:

1. t0,t1,t2 断开,t3 重连的客户;
2. t3 新来的客户; //ignore
3. t3 断开又重连的客户;

client 角度出发,理论上自己每次重连时访问 tinker 即可,但是访问量可能巨大. 通过把 vid 给后台,由后台来判断并告诉 client 是否需要访问 tinker.

## 逻辑实现

```sh

1. 每次connected后,客户端通过wss协议主动发送vid, 后台存储redis key HASH


'0': no need push after connect
'1': need push after connect

if wx:vid:xx key exsits and push_end and key.value is '1':
    push this client

set wx:vid:123xxx  '0'
set wx:peer:12131 'wx:vid:123xxxx'

2. pushed end时, push_end = True

3. disconnected时:

if push_end is not True:
    # 根据client.peer的value查找对应的vehicle id, 再修改响应的status 需要用到2组key,  peer:vid,  vid:status
    get wx:peer:1231  = wx:vid:123xxx
    set wx:vid:123xxx  '1'

del wx:peer:port

可能隐患: wx:vid:123xx 的k-v对会越来越多, 不过不是大问题.

```

## 新的问题

发现问题:

client ---nginx---websocket client 和 nginx 断开,没有通知到 web service

用手机模拟掉线的 client, 都飞行模式,但是 tcp 还是 established 的状态?

![Screenshot from 2018-12-18 18-44-03-85b610c3-d652-4558-af2e-4a3d703ce782](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202018-12-18%2018-44-03-85b610c3-d652-4558-af2e-4a3d703ce782.png)

看来并不能把 websocket 等同与 tcp:

```sh
TCP keepalive dont get passed through a web proxy. The websocket ping/pong will be forwarded by through web proxies. TCP
keepalive is designed to supervise a connection between TCP endpoints. Web socket endpoints is not equal to TCP endpoints.
A websocket connection can use several TCP connections between two websocket endpoints.
```

另外 nginx 里和 websocket 有关的超时设置, 是为了防止 nginx proxy websocket 的自动断开.

```sh
proxy_connect_timeout 5s;
proxy_send_timeout 15s;
proxy_read_timeout 86400s;
```

证明 proxy_read_timeout 是有影响的,但是也不能不要,否则 nginx 代理的 websocket 会断.

怎么办?看来还需要增加 1 层心跳功能. 考虑应用层加心跳或者 websocket 有自带的心跳 ping pong

研究 autobahn ,很简单就可以设置:

```sh
 global_factory.setProtocolOptions(autoPingInterval=5, autoPingTimeout=10)
```

而且 client 端一般都打开了 pong 功能,上述开启后, wireshark 可以轻松抓到 5s/次的心跳包

这样再再断网时,websocket 就能在 10s 内自动断开了.结合上面的机制,可以实现断线后仍然稳健 push.
