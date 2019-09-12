---
layout: post
title:
modified:
categories: Tech
tags: [wsl,win10,docker]
comments: true
---


<!-- TOC -->

- [laradock for win10](#laradock-for-win10)
    - [坑1 docker for windows volume目录](#坑1-docker-for-windows-volume目录)
    - [坑2 文件权限](#坑2-文件权限)

<!-- /TOC -->

# laradock for win10

决定在wsl下跑docker(laradock),docker server依赖`docker for windows`.

wsl的配置之前折腾过，主要是:
```
wsl-terminal
zsh
tmux
xlunch(方便copy和wsl下的linux gui在win10展示).
```

## 坑1 docker for windows volume目录 

不认`/mnt/d`这样的wsl.conf的默认配置目录. volume挂不上，自然build时各种fail。

法1: 手动改是把d盘直接mount在根目录.
```sh
alias mountD='sudo mount --bind /mnt/d/ /d'
```
每次开机都运行这个，麻烦,而且权限也是问题。

2. 修改wsl.conf
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
直接把window盘挂载到/上，这样wsl docker直接认了.

但是依赖`/mnt/c`的wsl-terminal不行，那就换掉好了，试了下发现了`terminus`这个神器。

## 坑2 文件权限

phpstorm在windows下修改文件,即便内容没变，文件都变成了755, 原因在于wsl.conf里的设置是umask=022.遂修改如下：
```sh
options = "metadata,dmask=022,fmask=133"
```
同时，让在wsl下新建的文件权限一致：(默认umask=000),在zsh里增加配置:
```sh
umask 022
```

这样在wsl git拉的代码，可以在wsl里改，也可以在windows通过ide改，没有权限问题，可以愉快的开发了。

