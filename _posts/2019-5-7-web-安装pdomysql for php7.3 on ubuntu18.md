---
layout: post
title:
modified:
categories: Tech
tags: [php]
comments: true
---

<!-- TOC -->

- [参考](#参考)

<!-- /TOC -->

# 参考

注:　很少直接用系统的php，一般是docker或直接用服务器的.还是安装1个

apt-get无法安装php-7.3的插件.
<https://kifarunix.com/how-to-install-php-7-3-3-on-ubuntu-18-04/>

参考这里方法后可以安装了
<https://my.oschina.net/u/2399303/blog/1512320>

```sh
# 安装php dev 不然就1个php 没有php-config phpize等
sudo apt-get install php7.3-dev;

# 下载安装包:
wget http://pecl.php.net/get/PDO_MYSQL-1.0.2.tgz

# 解压
tar zxvf PDO_MYSQL-1.0.2.tgz
cd PDO_MYSQL 

#生成configure
phpize

# 编译
./configure -with-php-config=/usr/bin/php-config -with-pdo-mysql=mysqlnd

```

最后发现可以直接安装完事儿..囧
```
sudo apt-get install php7.3-mysql
```