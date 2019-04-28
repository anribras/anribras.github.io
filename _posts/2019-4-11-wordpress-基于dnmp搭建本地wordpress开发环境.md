---
layout: post
title:
modified:
categories: Tech
 
tags: [wordpress]

comments: true
---

dnmp是docker搭建的php+mysql+nginx的环境.

<https://github.com/yeszao/dnmp.git>

`docker-compose up`运行后，需要改几个核心配置，才能运行wordpress,差点被坑死,记录下步骤:

1. 站点文件

放置到./www/下，命名为localhost，这个和docker-compose.yml的挂载路径相关;

2. 修改wp-config.php

这就是巨坑的点，mysql各种连不上，最后需要修改:
```php
/** MySQL主机 */
define('DB_HOST', 'mysql');
```

3. 导入备份数据库

phpMyAdmin的地址是localhost:8080;
备份并导入原来的wpdb就好.

4. nginx配置修改 

3点: 静态服务器和rewrite,和php:
```sh
#Rewite rules
#这个很重要，不然localhost报错
add_header 'Access-Control-Allow-Origin' 'http://localhost';
location /
{
        try_files $uri $uri/ /index.php?$args;
}
rewrite /wp-admin$ $scheme://$host$uri/ permanent;
# Static files
location ~ .*\.(js|css|woff2|otf)?$ {
        expires 7d;
}
location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|flv|mp4|ico)$ {
        expires 30d;
}
# php72 是php7.2运行的主机名字
location ~ \.php$ {
    fastcgi_pass   php72:9000;
    fastcgi_index  index.php;
    include        fastcgi_params;
    fastcgi_param  PATH_INFO $fastcgi_path_info;
    fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
}
```

注意到`mysql`,`php72`都是主机名字，这也是docker环境下配置需要注意的地方.

修改nginx配置后可以这样重启:
```sh
docker exec -it dnmp_nginx_1 nginx -s reload
```

5. 修改本机dns

修改`etc/hosts`，将127.0.0.1指向待调试的域名即可.

6. 关代理

不关代理，浏览器不生效.配合`SwitchyOmega`,切换到代理时，访问dep,切换到直连时，访问开发环境.

7. 分支

就用master分支，需要本地开发时，在根目录`touch .dev`即可.
需要区分的地方比如:
```sh
if (is_file(ABSPATH . '.dev')) {
    define('DB_HOST', 'mysql'); //Docker sql host named mysql
} else {
    define('DB_HOST', 'localhost'); //In LNMP deployment sql server from localhost.
}
```
