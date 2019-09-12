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
dnmp: <https://github.com/yeszao/dnmp>

## thinksns+

thinksns+: <https://slimkit.github.io/plus/guide/installation/install-plus.html>

按它的使用 docker composer,但是目录要改下.

这里的问题是在宿主环境使用 composer,但是我的宿主环境的 php 没有安装各种扩展.

讲道理是有问题的.　先试试在 php 容器里直接使用 composer 好了.

```sh
dphp72 即 docker exec -it dnmp_php72_1 /bin/bash
```

安装 composer:

```sh
curl -L https://getcomposer.org/composer.phar > /usr/local/bin/composer && \
chmod +x /usr/local/bin/composer && \
composer self-update
```

切换国内源:

```sh
composer config -g repo.packagist composer https://packagist.laravel-china.org
```

更新下载包:

```sh
composer update -vvv
```

如果下载中出错,运行`composer clear-cache` 重新来.

按教程继续.

缺少 ext-bcmatch:

```sh
docker-php-ext-install bcmath
```

重启 php-fpm 后好了。。。继续

```sh
docker restart xxx
```

```sh
 Illuminate\Database\QueryException  : SQLSTATE[HY000]: General error: 1215 Cannot add foreign key constraint (SQL: alter tabl
e `role_user` add constraint `role_user_user_id_foreign` foreign key (`user_id`) references `users` (`id`) on delete cascade on
update cascade)
```

迁移数据库时，表有问题?
不能建立外键，坑死了，找到原始可能是:

```sh
1）要关联的字段类型或长度不一致。
2）两个要关联的表编码不一样。
3）某个表已经有记录了。
4）将“删除时”和“更新时”都设置相同，如都设置成CASCADE。
```

发现 role_user 表的 user_id 定义的是 int(10),而其关联的主键是 bigint(20),遂改之..这个错误应该没了

- nginx
  没啥特殊的，站点在`plus/public`下

- storage
  给 storage 777 权限应该是要写

```sh
chomd 777 -R plus/storage
```
