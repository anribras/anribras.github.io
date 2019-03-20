---
layout: post
title:
modified:
categories: Tech
 
tags: [wordpress]

comments: true
---

<!-- TOC -->

- [nginx fastcgi cache](#nginx-fastcgi-cache)
    - [http](#http)
    - [server](#server)
    - [实测](#实测)

<!-- /TOC -->

cache是万金油，哪里不舒服抹哪里...

## nginx fastcgi cache

这个比wp任何cache插件都来的猛.


参考<https://www.daniao.org/3624.html>和<https://theshipyard.se/nginx-fastcgi-cache-for-wordpress/>


### http
```sh
fastcgi_cache_path /tmp/wpcache levels=1:2 keys_zone=WORDPRESS:250m inactive=1d max_size=1G;
fastcgi_temp_path /tmp/wpcache/temp;
fastcgi_cache_key "$scheme$request_method$host$request_uri";
fastcgi_cache_use_stale error timeout invalid_header http_500;
#忽略一切nocache申明，避免不缓存伪静态等
fastcgi_ignore_headers Cache-Control Expires Set-Cookie;
```
### server
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

        #新增的缓存规则
        fastcgi_cache_bypass $skip_cache;
        fastcgi_no_cache $skip_cache;
        add_header X-Cache "$upstream_cache_status From $host";
        fastcgi_cache WORDPRESS;
        add_header Cache-Control  max-age=0;
        add_header Last-Modified $date_gmt;
        fastcgi_cache_valid 200 301 302 1d;
}
```
### 实测

效果惊人...rqs从20多到1000多.但是要注意的就是skip_cache必须要生效:
```sh
cms后台不缓存;
登录用户访问不缓存;
动态查询不缓存;
POST不缓存;
```
另外地址本身必須是伪静态的，带index.php/xxx的肯定不会被缓存.

规则里不能有wordpress_xxx开头的cookie否则会被认为是系统登录相关的，所以需要在wp-config.php 里去掉:
```sh
define('TEST_COOKIE','myweb_');
```

第1次miss:

![Screenshot from 2019-03-20 13-24-33-b46a26a1-458e-49d2-a7ff-c0182f871bd2](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-03-20%2013-24-33-b46a26a1-458e-49d2-a7ff-c0182f871bd2.png)

第2次hit:

![Screenshot from 2019-03-20 13-25-10-6d026425-5523-4ee7-969d-ca0d9a3cdfec](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-03-20%2013-25-10-6d026425-5523-4ee7-969d-ca0d9a3cdfec.png)

已登录用户则直接bypass，不通过fastcgi缓存:

![Screenshot from 2019-03-20 13-27-04-b5906215-586c-4238-864f-a7b41f7b71ff](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-03-20%2013-27-04-b5906215-586c-4238-864f-a7b41f7b71ff.png)
