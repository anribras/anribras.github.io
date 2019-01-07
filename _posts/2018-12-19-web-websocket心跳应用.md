---
layout: post
title:
modified:
categories: Tech
 
tags: [python]

comments: true
---

<!-- TOC -->

- [User story](#user-story)
- [不足分析](#不足分析)
- [逻辑实现](#逻辑实现)
- [新的问题](#新的问题)

<!-- /TOC -->


websocket 稳定push设计方案


## User story

1000台客户端同时在线, 突然发现bug, 导致大面积瘫痪, server使用websocket做更新推送.

版本紧急修复后,上线实现步骤:

1. 修复版本
2. admin 登录portal  
3. 上传当前补丁信息,  设置发布条件: push时间, 车机id, 地理限制, 人数限制, push频率...等等. 
4. 设置完成后, 页面显示push目标人数, 即当前补丁正式push前的在线人数. 
5. push操作,点击后开始执行push, 后台按设定策略开始通知在线车机. 

当前实现,可以解决在push期间出现client异常掉线/新用户上线的场景.

1.  用python list  存储 [ client object1 ,... ],client增加属性flag=0,
2.  push时,对每个client发送命令. 发送完毕的client flag=1.
    循环检测list里dict元素的flag,直到最后1个flag不为0,即所有client被push了,才push end.
    push期间, 再次push时 trylock
    push期间 ,重连上线或者新上线的client 可以处理(client会append 到list)
    push期间 ,如果client掉线, remove(client).

## 不足分析

上线步骤4和5期间, 考虑可能发生在以下任意时间段(t0,t1,t2,t3)的掉线/重新上线:

---t0 ----- tinker ready --- t1 ---push start ---t2--- push end -----t3------

t0之后的新用户 由app自己处理;
在t2之前因为掉线而重新上线的,都由上面已经实现的机制处理;
即主要考虑在t0/t1/t2掉线,但在t3又重连的客户,如何识别?

t3连上的客户可能包括已下3种:

1. t0,t1,t2断开,t3重连的客户; 
2. t3新来的客户; //ignore
3. t3断开又重连的客户;


client角度出发,理论上自己每次重连时访问tinker即可,但是访问量可能巨大. 通过把vid给后台,由后台来判断并告诉client是否需要访问tinker.

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

client ---nginx---websocket   client和nginx断开,没有通知到web service 


用手机模拟掉线的client, 都飞行模式,但是tcp还是established的状态?

![Screenshot from 2018-12-18 18-44-03-85b610c3-d652-4558-af2e-4a3d703ce782](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202018-12-18%2018-44-03-85b610c3-d652-4558-af2e-4a3d703ce782.png)

看来并不能把websocket等同与tcp:
```
TCP keepalive dont get passed through a web proxy. The websocket ping/pong will be forwarded by through web proxies. TCP 
keepalive is designed to supervise a connection between TCP endpoints. Web socket endpoints is not equal to TCP endpoints.
A websocket connection can use several TCP connections between two websocket endpoints.
```

另外nginx里和websocket有关的超时设置, 是为了防止nginx proxy websocket的自动断开.

```
proxy_connect_timeout 5s;
proxy_send_timeout 15s;
proxy_read_timeout 86400s;  
```
证明proxy_read_timeout是有影响的,但是也不能不要,否则nginx代理的websocket会断.

怎么办?看来还需要增加1层心跳功能. 考虑应用层加心跳或者websocket有自带的心跳ping pong

研究autobahn ,很简单就可以设置:

```
 global_factory.setProtocolOptions(autoPingInterval=5, autoPingTimeout=10)
```
而且client端一般都打开了pong功能,上述开启后, wireshark可以轻松抓到5s/次的心跳包

这样再再断网时,websocket就能在10s内自动断开了.结合上面的机制,可以实现断线后仍然稳健push.

