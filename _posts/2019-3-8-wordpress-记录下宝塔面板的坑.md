---
layout: post
title:
modified:
categories: Tech
 
tags: [wordpress]

comments: true
---

<!-- TOC -->

- [云服务器](#云服务器)
- [bt_panel](#bt_panel)
    - [问题１　php安装失败](#问题１　php安装失败)
    - [问题2 ftp失败](#问题2-ftp失败)
    - [wordpress升级失败](#wordpress升级失败)
    - [安装WP-DBManager报错](#安装wp-dbmanager报错)

<!-- /TOC -->

## 云服务器

阿里云搞活动，香港清凉云下手晚了，买了1个204一年的新加坡清凉云.速度将就，电信的可能还不如我的bwg.

出于放心，先把云盾什么的拿掉.

自然是lnmp的环境安装了，都上最新的版本就好.

## bt_panel

选择了bt_panel，然后升级到最新的6.8.9.可以破解掉...先不着急

### 问题１　php安装失败
在面板设置里增大swap为2G

### 问题2 ftp失败

在云控制台，放行`20,21,39000-40000`端口，最后１组是fpt passive模式时宝塔里设置的.

有意思的是，尽管bt在iptables里已经放行了，但是端口的总闸还是在控制台的.

### wordpress升级失败

```sh
chattr -i /www/wwwroot/149.129.56.27/.user.ini
chown -R www:root /www/wwwroot  
```
这里是给文件夹nginx默认用户www访问的权限.

### 安装WP-DBManager报错

![Screenshot from 2019-03-13 18-08-00-a2943104-f9e4-418a-baca-d6c8e93e8ad0](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-03-13%2018-08-00-a2943104-f9e4-418a-baca-d6c8e93e8ad0.png)


openbase_dir设置的问题:
<https://www.hacksparrow.com/wp-dbmanager-error-mysql-dump-path-does-not-exist-please-check-your-mysqldump-path-under-db-options.html>
        
<https://blog.csdn.net/fdipzone/article/details/54562656>
        