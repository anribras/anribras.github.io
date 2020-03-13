---
layout: post
title:
modified:
categories: Tech

tags: [wordpress]

comments: true
---

<!-- TOC -->

- [http 缓存字段](#http-缓存字段)
  - [Last-Modified](#Last-Modified)
  - [ETag](#ETag)
  - [Expires](#Expires)
  - [Cache-Control](#Cache-Control)
  - [用法](#用法)
- [nginx fastcgi cache](#nginx-fastcgi-cache)
  - [nginx http 配置](#nginx-http-配置)
  - [nginx server 配置](#nginx-server-配置)
  - [实测](#实测)
  - [对比下加和不加的效果](#对比下加和不加的效果)
  - [疑问 1](#疑问-1)

<!-- /TOC -->

cache 是万金油，哪里不舒服抹哪里...

```sh
客户端缓存;
server端页面缓存;
server端静态资源缓存
server端数据库缓存;
```

## http 缓存字段

http 缓存通过 http 协议,让客户端于服务端相互配合.

<https://blog.csdn.net/shadow_zed/article/details/81268475>

http 控制缓存的字段:

### Last-Modified

```sh
server, Last-Modified: Wednesday, 20-Mar-2019 05:55:18 GMT
client, If-Modified-Since: Wednesday, 20-Mar-2019 05:38:51 GMT
```

1. server(webserver,或 web 应用服务器)告诉 client(浏览器,app 等)该资源上次的更新时间 Last-Modified, status 200.
2. client 第 2 次请求时,缓存资源,并保存该 time;
3. client 第 2 次请求时,If-Modified-Since 带上 time，服务端比较，如果 time 没变，返回 304,让 client 使用缓存.否则返回新资源,status 200.

### ETag

```sh
server: ETag: W/"5c80a8a0-629a"
client: If-None-Match:
```

1. 类似 last-modified-time,服务器给缓存资源打标签，客户端访问时用 If-None-Match 带上标签，服务端比对，没变化返回 304,有变化返回新资源 200.

和 Last-Modified 比较，Etag 在分布式里不要用，每台机器的 etag 不一样，策略会乱.

### Expires

```sh
server, Expires: Wednesday, 29-Mar-2019 05:55:18 GM
```

还是类似的，server 告诉 client1 个绝对时间，在 expire time 之前，客户端都可以直接用自己的缓存，不必访问服务器.(不过要求是 server 与 client 时间需要严格同步)

Last-Modified 要求 client 至少还要请求 1 次，(即回 304 的这趟), Expired 则告诉客户端，你 1 次请求也不需要了.

### Cache-Control

类似 Expired,但 max-age 指定的相对时间，即:请求返回时间的`timestamp + max-age指定的秒数`为缓存的有效期.

同时设置 Expired, Last-Modified,Cache-Control max-age 等,Cached-Control 会覆盖前者.

```sh
# 长期的稳定资源， server让client使用客户端缓存就好，无需访问服务器;
Cache-Control: max-age: 31536000
# 不是不让client缓存，而是使用缓存前，先需要让服务器做1次检查.
Cache-Control: private、must-revalidate, no-cache
# server让client每次都请求新是
Cache-Control: max-age=0
# server让client别缓存该资源，真正的别缓存，用的较少.
Cache-Control: no-store
```

### 用法

成熟的 web 服务器框架下，当缓存过期时，并不是粗暴的直接请求那么简单.可以通过:

1. 新资源更新`版本号`.

即每次更新 js,css 后，同时给他们加上版本号.

2. 资源路径带 md5,文件变化，则路径变化

由此引申的前端自动化构建框架，gulp,再到 wpgulp<https://github.com/ahmadawais/wpgulp/>..很牛逼了.

<https://blog.csdn.net/lqlqlq007/article/details/79940017>

max-age 正确策略:

```sh
1 变化较少的HTML使用每次服务端验证的方式(304)来保证资源是最新的;
2 经常变化的CSS和JS则可以使用设置max-age，但发生变更后更新资源路径（如重新计算文件的哈希，并把哈希值加入文件名中）的方式来保证资源是最新的，当然，这样做需要在HTML中同步更新依赖CSS和JS的资源路径（虽然之前的CSS和JS仍在缓存期内，但实际页面已经正确使用了更新后的资源）。
```

## nginx fastcgi cache

参考<https://www.daniao.org/3624.html>和<https://theshipyard.se/nginx-fastcgi-cache-for-wordpress/>

cached 如果在 nginx 里做，就应该在 nginx 里配置，比如 fastcgi cached.

如果是 web 应用服务器做，可以在 html header 里加 http 的字段.

但是 nginx 自然比 php 里的一票 cache 插件效率来的高，再配合`nginx helper`这个插件,值得一试.

配置如下:

### nginx http 配置

```sh
fastcgi_cache_path /tmp/wpcache levels=1:2 keys_zone=WORDPRESS:250m inactive=1d max_size=1G;
fastcgi_temp_path /tmp/wpcache/temp;
fastcgi_cache_key "$scheme$request_method$host$request_uri";
fastcgi_cache_use_stale error timeout invalid_header http_500;
#忽略一切nocache申明，避免不缓存伪静态等
fastcgi_ignore_headers Cache-Control Expires Set-Cookie;
```

注，缓存放在内存也是可以的,如/var/run/wpcache.

### nginx server 配置

```sh
location ~ [^/]\.php(/|$)
{
    set $skip_cache 0;
    #post访问不缓存
    if ($request_method = POST) {
    set $skip_cache 1;
    }
    #动态查询不缓存
    if ($query_string != "") {
    set $skip_cache 1;
    }
    #后台等特定页面不缓存（其他需求请自行添加即可）
    if ($request_uri ~* "/wp-admin/|/xmlrpc.php|wp-.*.php|/feed/|index.php|sitemap(_index)?.xml") {
    set $skip_cache 1;
    }
    #对登录用户、评论过的用户不展示缓存
    if ($http_cookie ~* "comment_author|wordpress_[a-f0-9]+|wp-postpass|wordpress_no_cache|wordpress_logged_in") {
    set $skip_cache 1;
    }
    try_files $uri =404;
    fastcgi_pass  unix:/tmp/php-cgi-73.sock;
    fastcgi_index index.php;
    include fastcgi.conf;
    include pathinfo.conf;
    ## fastcgi cache

    #依据上面的缓存规则
    fastcgi_cache_bypass $skip_cache;
    fastcgi_no_cache $skip_cache;
    # 命令状态
    add_header X-Cache "$upstream_cache_status From $host";
    fastcgi_cache WORDPRESS;
    # 告诉客户端,该缓存资源在客户端的存活时间，max-age=0，表示客户端不应该缓存.
    add_header Cache-Control  max-age=0;
    # 告诉客户端，当前的资源是最新的，也就是客户端应该使用该缓存资源.
    add_header Last-Modified $date_gmt;
    fastcgi_cache_valid 200 301 302 1d;

}
```

### 实测

要注意的就是 skip_cache 规则,也就是哪些情况不缓存:

```sh
cms后台不缓存;
登录用户访问不缓存;
动态查询不缓存;
POST不缓存;
```

另外地址本身必須是伪静态的，带 index.php/xxx 的肯定不会被缓存.

规则里不能有 wordpress_xxx 开头的 cookie 否则会被认为是系统登录相关的，所以需要在 wp-config.php 里去掉:

```sh
define('TEST_COOKIE','myweb_');
```

某篇文章，第 1 次 miss:

![Screenshot from 2019-03-20 13-24-33-b46a26a1-458e-49d2-a7ff-c0182f871bd2](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-03-20%2013-24-33-b46a26a1-458e-49d2-a7ff-c0182f871bd2.png)

第 2 次 hit:

![Screenshot from 2019-03-20 13-25-10-6d026425-5523-4ee7-969d-ca0d9a3cdfec](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-03-20%2013-25-10-6d026425-5523-4ee7-969d-ca0d9a3cdfec.png)

已登录用户则直接 bypass，不通过 fastcgi 缓存:

![Screenshot from 2019-03-20 13-27-04-b5906215-586c-4238-864f-a7b41f7b71ff](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-03-20%2013-27-04-b5906215-586c-4238-864f-a7b41f7b71ff.png)

### 对比下加和不加的效果

加 fastcgi cache

```sh
➜  conf ab -n 2000 -c 500 http://149.129.56.27/local/256/
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 149.129.56.27 (be patient)
Completed 200 requests
Completed 400 requests
Completed 600 requests
Completed 800 requests
Completed 1000 requests
Completed 1200 requests
Completed 1400 requests
Completed 1600 requests
Completed 1800 requests
Completed 2000 requests
Finished 2000 requests


Server Software:        nginx
Server Hostname:        149.129.56.27
Server Port:            80

Document Path:          /local/256/
Document Length:        12954 bytes

Concurrency Level:      500
Time taken for tests:   7.796 seconds
Complete requests:      2000
Failed requests:        0
Write errors:           0
Total transferred:      26486000 bytes
HTML transferred:       25908000 bytes
Requests per second:    256.53 [#/sec] (mean)
Time per request:       1949.090 [ms] (mean)
Time per request:       3.898 [ms] (mean, across all concurrent requests)
Transfer rate:          3317.60 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0  390 984.8      0    4082
Processing:     0  763 1586.4     12    7750
Waiting:        0  458 1264.1     11    6408
Total:          0 1153 2139.4     12    7756

Percentage of the requests served within a certain time (ms)
  50%     12
  66%    208
  75%    896
  80%   1884
  90%   5652
  95%   6407
  98%   6427
  99%   6427
 100%   7756 (longest request)
```

不加 fastcgi cache

```sh
➜  conf ab -n 200 -c 50 http://149.129.56.27/local/256/
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 149.129.56.27 (be patient)
Completed 100 requests
Completed 200 requests
Finished 200 requests


Server Software:        nginx
Server Hostname:        149.129.56.27
Server Port:            80

Document Path:          /local/256/
Document Length:        12952 bytes

Concurrency Level:      50
Time taken for tests:   10.264 seconds
Complete requests:      200
Failed requests:        0
Write errors:           0
Total transferred:      2626000 bytes
HTML transferred:       2590400 bytes
Requests per second:    19.49 [#/sec] (mean)
Time per request:       2565.880 [ms] (mean)
Time per request:       51.318 [ms] (mean, across all concurrent requests)
Transfer rate:          249.86 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    4   4.1      1      11
Processing:    79 2458 765.5   2701    3743
Waiting:       77 2458 765.6   2701    3743
Total:         79 2462 766.6   2711    3744

Percentage of the requests served within a certain time (ms)
  50%   2711
  66%   2773
  75%   2841
  80%   2946
  90%   3362
  95%   3413
  98%   3633
  99%   3674
 100%   3744 (longest request)
```

少说 20 倍性能提升...

### 疑问 1

客户端发送 no-cache 时，意思是客户端不想要缓存资源，但是 nginx 仍然返回了 HIT. nginx fastcgi 是如何处理客户端发送的 cached-control no cache 请求的? 不处理 or 直接发送到 wep app server?
![Screenshot from 2019-04-04 10-21-17-137d19ac-2ebf-4e2a-b6e9-96738f16a82b](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-04-04%2010-21-17-137d19ac-2ebf-4e2a-b6e9-96738f16a82b.png)
