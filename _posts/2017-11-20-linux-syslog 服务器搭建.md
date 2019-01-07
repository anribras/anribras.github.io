---
layout: post
title:
modified:
categories: Tech
 
tags: [backend,linux]

  
comments: true
---
<!-- TOC -->

- [前言](#前言)
- [syslog](#syslog)
- [配置](#配置)
- [使用](#使用)

<!-- /TOC -->

### 前言
一直想把项目里的log dump到网络上，用来远程客户的故障，syslog可以满足所有需求。
正好搭了bwg来学习，决定部署syslog到服务器，接收来自项目的log。

### syslog
分为服务端和客户端，运行syslog的机器既可以是device，relay，也可以是acceptor。很多文章讲的非常清楚，我姑且引用下好了。

[python syslog](https://www.cnblogs.com/newguy/p/6093290.html)

[embedded linux](http://blog.csdn.net/yangxuan12580/article/details/51497069)

[rsyslog 1](https://huoding.com/2014/05/09/347)

[rsyslog 2](http://www.cnblogs.com/tobeseeker/archive/2013/03/10/2953250.html)


### 配置

把/etc/rsyslog.conf中的以下２行注释打开，即打开了远程log功能:
使用UDP port=514来监控远程日志，也可以配置TCP的。
```sh
#provides UDP syslog reception
module(load="imudp")
input(type="imudp" port="514")
```
重启服务 
```
systemctl restart syslog.service
```

### 使用
最简单的方式，在远程机器使用`logger`命令:
```
logger "this is a logger test from remote bravo." -n 65.49.213.93 -P 514
```

然后在远程syslog服务器就可以看到远程的日志了:
```
root@anribras:/var/log# cat /var/log/syslog | tail
Dec 20 20:10:11 ThinkPad-X260 bravo[16541] this is a logger test from remote bravo.
```

在项目里不管是c/c++/python/java，android/ios...都有办法使用syslog。如果不只希望自己的程序往syslog填，而是整个终端的都往里去怎么办呢?

[这篇文章](http://urbanautomaton.com/blog/2014/09/09/redirecting-bash-script-output-to-syslog/)解释了一个命令可以将终端输出直接导入到syslog。
```
exec 1> >(logger -s -t $(basename $0)) 2>&1
```

我稍微改下，不加tag,(`-t`),使用远程服务器的命令:

```
exec 1> >(logger -s -n 65.49.213.93 -P 514) 2>&1
```
这样整个终端的内容都过去了，不想用的时候关闭终端即可，非常方便。

以上仅仅是皮毛，rsyslog是非常强大的，后续再研究进阶的内容。
