---
layout: post
title:
modified:
categories: Tech
tags: [wsl, win10, docker]
comments: true
---

<!-- TOC -->

- [laradock for win10](#laradock-for-win10)
- [坑 1 docker for windows volume 目录](#坑-1-docker-for-windows-volume-目录)
- [坑 2 文件权限](#坑-2-文件权限)

<!-- /TOC -->

## laradock for win10

决定在 wsl 下跑 docker(laradock),docker server 依赖`docker for windows`.

wsl 的配置之前折腾过，主要是:

```sh
wsl-terminal
zsh
tmux
xlunch(方便copy和wsl下的linux gui在win10展示).
```

## 坑 1 docker for windows volume 目录

不认`/mnt/d`这样的 wsl.conf 的默认配置目录. volume 挂不上，自然 build 时各种 fail。

法 1: 手动改是把 d 盘直接 mount 在根目录.

```sh
alias mountD='sudo mount --bind /mnt/d/ /d'
```

每次开机都运行这个，麻烦,而且权限也是问题。

修改 wsl.conf:

```sh
#Let’s enable extra metadata options by default
[automount]
enabled = true
root = /
options = "metadata,umask=022"
mountFsTab = false

#Let’s enable DNS – even though these are turned on by default, we’ll specify here just to be explicit.
[network]
generateHosts = true
generateResolvConf = true

#All windows program shoulbe be normally run in wsl. great!
[interop]
enable = true
appendWindowsPath = true
```

直接把 window 盘挂载到/上，这样 wsl docker 直接认了.

但是依赖`/mnt/c`的 wsl-terminal 不行，那就换掉好了，试了下发现了`terminus`这个神器。

## 坑 2 文件权限

phpstorm 在 windows 下修改文件,即便内容没变，文件都变成了 755, 原因在于 wsl.conf 里的设置是 umask=022.遂修改如下：

```sh
options = "metadata,dmask=022,fmask=133"
```

同时，让在 wsl 下新建的文件权限一致：(默认 umask=000),在 zsh 里增加配置:

```sh
umask 022
```

这样在 wsl git 拉的代码，可以在 wsl 里改，也可以在 windows 通过 ide 改，没有权限问题，可以愉快的开发了。
