---
layout: post
title:
modified:
categories: Tech
 
tags: [wordpress]

comments: true
---

<!-- TOC -->

- [前端性能优化](#前端性能优化)
- [数据库优化.](#数据库优化)
- [wordpress系统](#wordpress系统)
- [代码级别优化](#代码级别优化)
- [精简主题](#精简主题)
- [基于nginx优化](#基于nginx优化)
- [最终方案](#最终方案)

<!-- /TOC -->

## 前端性能优化
```sh
合并css js, 
子域名分担负载.
cdn
```

## 数据库优化.
```sh
- innodb引擎
- 合并表，减少慢查询什么的
- wordpress transient api类似数据库缓存，可了解一下:
    https://codex.wordpress.org/Transients_API
```

## wordpress系统
```sh
减少post_revision
插件: 
    - 少用不必要插件
    - 合并插件
缓存:
    - w3 total cache　＋　opcode + memcached.
    - nginx fastcgi cache ,配合1个插件叫nginx helper.做page cached
内存优化,可到bt里直接设置:
    - https://www.vpsss.net/4796.html
图像转化引擎:
    - imagemagick引擎，可用本地command，也可以用php的扩展
```
## 代码级别优化
```sh
rest精简查询,减少不相关信息输出--->graph ql也是方向
图片延迟加载lazysizes;
ajax
```
## 精简主题

主题重构,coxy还是太重了，而且functions.php加密了


## 基于nginx优化

参考:
<https://www.nginx.com/blog/9-tips-for-improving-wordpress-performance-with-nginx/>


## 最终方案

1. gulp做前端框架，什么压缩，合并都在一起了,不用插件

2. fastcgi cache做页面缓存，处理不需要经过php, 理论上是要快过 w3tc, wsc这些插件
<https://www.digitalocean.com/community/questions/how-can-i-configure-w3-total-cache-fastcgi-and-optimize-nginx-for-best-performance-on-ubuntu-14-04-x32-10>
 优点:快，缺点:缓存管理.
