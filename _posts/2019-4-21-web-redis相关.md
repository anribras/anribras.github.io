---
layout: post
title:
modified:
categories: Tech
tags: [redis]
comments: true
---

<!-- TOC -->

- [缓存击穿](#缓存击穿)
- [缓存雪崩](#缓存雪崩)

<!-- /TOC -->


## 缓存击穿

redis也无，查sql也无，那后1次查询就是多余的嘛

解决:

1.布隆过滤器，也就是建立一个筛选器，拦截那些可能为空的到sql的查询

2.空的sql查询也缓存，这样就不会把负担通到sql
## 缓存雪崩
访问redis的key突然失效(过期)，大量重复的访问全部涌向sql,导致数据库崩溃

解决思路:

1.那就不要让key-v随便失效,增加个sign, sign先过期，如果检测到sign过期则更新cache
```sh
key-v-sign 10s 过期
key-v 20s过期 
```

2.给访问上锁，生效未恢复前，禁止其他访问，其他等待用户可能一直等待，不太好

3.定期恢复检查，并恢复缓存.