---
layout: post
title:
modified:
categories: Tech
 
tags: [wordpress]

comments: true
---


<!-- TOC -->

- [安装xdebug](#安装xdebug)

<!-- /TOC -->

还是基于宝塔面板，很遗憾php7.3还没看到这个扩展,只有自己来了.

参考官网的就成<https://xdebug.org/wizard.php>

## 安装xdebug
```sh
wget http://xdebug.org/files/xdebug-2.7.0.tgz
tar zxvf xdebug-2.7.0.tgz
cd xdebug-2.7.0
```
检查输出
```sh
 /www/server/php/73/bin/phpize
```
应该为:
```sh
Configuring for:
PHP Api Version:         20180731
Zend Module Api No:      20180731
Zend Extension Api No:   320180731
```
然后编译:
```sh
./configure --with-php-config=/www/server/php/73/bin/php-config
make
cp modules/xdebug.so /www/server/php/73/lib/php/extensions 
```
修改php.ini,增加:
```sh
[Xdebug]
zend_extension = xdebug.so
xdebug.auto_trace=on
xdebug.collect_params=on
xdebug.collect_return=on
xdebug.trace_output_dir="/tmp/xdebug"
xdebug.profiler_enable=on
xdebug.profiler_output_dir="/tmp/xdebug"
xdebug.remote_enable = on
xdebug.remote_handler = dbgp
#因为自己的php也是用的vps上的,这里填本机的ip
xdebug.remote_host= localhost
xdebug.remote_port = 9000
xdebug.idekey = PHPSTORM
```

然后各种不行...找了下原因,单连的时候，也就不是DBGp proxy时，server是在phpstorm上，vps不可能连到我自己的电脑上.


跑程序的实体php是xdebug的server 监听9000.

ide的debug调试器是client, 连上server接收调试信息.

IDE需要配置1个php环境，最好是本地的

