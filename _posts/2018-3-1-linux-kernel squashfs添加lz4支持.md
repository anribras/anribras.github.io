---
layout: post
title:
modified:
categories: Tech
excerpt: 
tags: [linux,squashfs]

  
comments: true
---
<!-- TOC -->

- [背景](#背景)
- [添加suquashfs lz4 support](#添加suquashfs-lz4-support)
- [SchedAutogroup支持](#schedautogroup支持)

<!-- /TOC -->

### 背景

FOTA开发中，考虑到升级文件的颗粒度小的问题，将squashfs打包挪到车机上,传输期间可以做到任意小的文件，做到真正的差分升级。

```
-all-root -noappend -comp lzo -Xcompression-level 9
```
发现`lzo`特别的慢，尝试用`lz4`，倒是快了不少，悲剧的是，后续mount不上去，
查阅资料知道默认kernel3.11以上才有squashfs lz4 support,谁让我们的low cost方案才是kernel 3.4.5呢。

撸起袖子开始干，修改内核squashfs部分增加lz4支持。

### 添加suquashfs lz4 support

kernel要改两部分，一部分是`lib/lz4`相关源码及Kconfig,Makefile,一部分是`fs/squashfs`下的lz4 wrapper.

[lz4 wrapper patch 参考](https://patchwork.kernel.org/patch/2830988/)

[lib 参考](https://git.kernel.org/pub/scm/linux/kernel/git/pkl/squashfs-lz4.git/snapshot/squashfs-lz4-master.tar.gz)

`make menuconfig`时，需要打开相关选项:

![2018-03-01-18-35-19](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-03-01-18-35-19.png)

另外还需要添加2个头文件
```
include/linux/lz4.h
include/linux/decompress/unlz4.h
```

最后make成功，替换Image后，真的可以用，小激动一个，绝对是独门solution。

### SchedAutogroup支持

mksquash 在车机端太吃cpu,busybox 的nice好像没有效果。参考了下[https://serverfault.com/questions/405092/nice-level-not-working-on-linux] 发现连这个选项都没有，尝试在内核里添加AU

算了动kernel 太复杂 直接下个cpulimit搞定



