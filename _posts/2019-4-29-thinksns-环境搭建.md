---
layout: post
title:
modified:
categories: Tech
tags: [thinksns]
comments: true
---


## dnmp

这个绝的挺好用的


## thinksns+

* composer

<https://github.com/yeszao/dnmp>

按它的使用docker composer,但是目录要改下.

这里的问题是在宿主环境使用composer,但是我的宿主环境的php没有安装各种扩展.

讲道理是有问题的.　先试试在php容器里直接使用composer好了.

```
curl -L https://getcomposer.org/composer.phar > /usr/local/bin/composer && \
chmod +x /usr/local/bin/composer && \
composer self-update
```
有又错:
```sh
[ErrorException]                                                                                                              
  zlib_decode(): data error 
```
<https://stackoverflow.com/questions/33501240/composer-error-while-installing-laravel-failed-to-decode-response-zlib-decode>
运行`composer clear-cache` 重新来.

缺少ext-bcmatch:

```sh
docker-php-ext-install bcmath
```

重启php-fpm后好了。。。继续
```sh
 Illuminate\Database\QueryException  : SQLSTATE[HY000]: General error: 1215 Cannot add foreign key constraint (SQL: alter tabl
e `role_user` add constraint `role_user_user_id_foreign` foreign key (`user_id`) references `users` (`id`) on delete cascade on 
update cascade) 
```
迁移数据库时，表有问题?



