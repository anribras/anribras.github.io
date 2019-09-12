---
layout: post
title:
modified:
categories: Tech
excerpt:
tags: [linux, squashfs]

comments: true
---

<!-- TOC -->

- [背景](#背景)
- [添加 suquashfs lz4 support](#添加-suquashfs-lz4-support)
- [SchedAutogroup 支持](#SchedAutogroup-支持)

<!-- /TOC -->

### 背景

FOTA 开发中，考虑到升级文件的颗粒度小的问题，将 squashfs 打包挪到车机上,传输期间可以做到任意小的文件，做到真正的差分升级。

```
-all-root -noappend -comp lzo -Xcompression-level 9
```

发现`lzo`特别的慢，尝试用`lz4`，倒是快了不少，悲剧的是，后续 mount 不上去，
查阅资料知道默认 kernel3.11 以上才有 squashfs lz4 support,谁让我们的 low cost 方案才是 kernel 3.4.5 呢。

撸起袖子开始干，修改内核 squashfs 部分增加 lz4 支持。

### 添加 suquashfs lz4 support

kernel 要改两部分，一部分是`lib/lz4`相关源码及 Kconfig,Makefile,一部分是`fs/squashfs`下的 lz4 wrapper.

[lz4 wrapper patch 参考](https://patchwork.kernel.org/patch/2830988/)

[lib 参考](https://git.kernel.org/pub/scm/linux/kernel/git/pkl/squashfs-lz4.git/snapshot/squashfs-lz4-master.tar.gz)

`make menuconfig`时，需要打开相关选项:

![2018-03-01-18-35-19](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-03-01-18-35-19.png)

另外还需要添加 2 个头文件

```
include/linux/lz4.h
include/linux/decompress/unlz4.h
```

最后 make 成功，替换 Image 后，真的可以用，小激动一个，绝对是独门 solution。

### SchedAutogroup 支持

mksquash 在车机端太吃 cpu,busybox 的 nice 好像没有效果。参考了下[这个](https://serverfault.com/questions/405092/nice-level-not-working-on-linux) 发现连这个选项都没有，尝试在内核里添加 AU

算了动 kernel 太复杂 直接下个 cpulimit 搞定
