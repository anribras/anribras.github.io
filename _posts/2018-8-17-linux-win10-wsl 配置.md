---
layout: post
title:
modified:
categories: Tech

tags: [linux]

comments: true
---

<!-- TOC -->

- [安装系统](#安装系统)
- [win10](#win10)
  - [wsl](#wsl)
  - [bash 优化](#bash-优化)
  - [Xlaunch](#Xlaunch)
  - [配色](#配色)
  - [wsl 权限问题](#wsl-权限问题)

<!-- /TOC -->

新购置了 Sangsum 860evo 512G ssd,打算替换原笔记本机械盘.

机器: Thinkpad X260, 单机械硬盘 512G, 已安装 win10+ubuntu16.04.

保底目标: 无缝迁移原 hdd 到 ssd 上,特别 ubuntu16.04 作为工作环境,不能有影响.

更精致的目标: windows 重装, ubuntu 个人部分到 ssd, 工作部分到外接的 hdd(当移动硬盘用咯),这样插上盒子就是工作环境, 拔掉盒子就是自己的个人环境.

如果 hdd 外接硬盘盒子能直接使用,那就可在新 ssd 上随意折腾...只要不影响原工作.

### 安装系统

1. 启动盘准备.
   用 refus 制作 win10 ios 镜像即可

2. ssd 分区 for win10
   分区助手, 4k 对齐.系统盘 60G. Work 盘 150G.

3. 拆换硬盘,用 win10 启动盘安装 win10.

4. 原机械盘外接到硬盘盒子; ubuntu 启动盘也插上;

机械盘外接能使用进原 ubuntu.完美.不用重新装 ubuntu 了.

至此，新 win10 运行在 ssd 上，原工作环境运行在外接的 hhd 硬盘盒子.

### win10

基于`1803 iso`,其他软件该装的都装,注意有个叫`神龙`的系统激活软件最好用.

#### wsl

以前就听过，理论上的完美双系统, 必须试验下.

en...体验各方面介于 pure linux 和 virtual machine 之间.

#### bash 优化

基于叫`wsl-open`的折腾，wsl 自带的 bash 太丑了.

tmux+vim 的环境怎能错过?按自己的以前的 config 配置一遍... 难点自然是`vim youcompleteme`了.

tmux 原来的 tmux.conf 不能直接用，一行一行实验最后写了个新的.

遇到一个大坑是从 bash 到 windows 的 clipboard 问题，网上有牛人来解决:
[这里](https://github.com/Microsoft/WSL/issues/892),总体是下载一个`XLaunch`的东西,另外 vim 需要编译`+clipboard`的支持,还是挺麻烦的

#### Xlaunch

不装不知道，这东西非常牛，直接将 x11 的界面弄到 win10 上了，使得在 wsl 开发 GUI 也是可能，比如我正在做的这个 cross-platform displaySim.

#### 配色

这个折腾我太久了，配色是个大问题.找了半天发现 codesschool 还可以，再修改下 Green bold 的配色，文件夹也看的清楚了:

```sh
mintty theme:base16-codeschool.minttyrc
oh-my-zsh: risto
```

先这样吧,折腾的时间耗不起:

markdown 自然还是用 vscode 来写，原来的七牛截图不能用，把原来的七牛配置稍微改下就好:
![2018-08-20-15-15-53](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-08-20-15-15-53.png)

working bash 最终效果:

![2018-08-22-14-51-30](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-08-22-14-51-30.png)

#### wsl 权限问题

美好的故事是把文件都房子放在 window 盘，wsl mount 在了/mnt/x/下,尽情用之...但发现所有文件的权限都成了 777.

比如原 linux git 仓库的文件在在 windows/project 下，再用 wsl 去读时，全是 modified 的状态...

wsl 固有的文件系统方式，解释[在这里](https://github.com/Microsoft/WSL/issues/2182)

这也是 1 坑，先记在这里.

这个貌似是[解决方案](https://blogs.msdn.microsoft.com/commandline/2018/01/12/chmod-chown-wsl-improvements/)

```sh
sudo umount /mnt/c
sudo mount -t drvfs C: /mnt/c -o metadata,uid=1000,gid=1000,umask=22,fmask=111
```

不试不知道...原来之前的文件夹被加背景色，就是因为权限加成了 777! 改了之后，发现 mintty theme 都能看了.没必要去各种尝试了..

[参考](https://docs.microsoft.com/en-us/windows/wsl/wsl-config),增加`/etc/wsl.conf`,不用每次都手动:

```ini
# Let’s enable extra metadata options by default
[automount]
enabled = true
root = /mnt/
options = "metadata,umask=22"
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

需要注意在 wsl 下， 文件权限都是 755, 特别是看 git 在 windows 下的文件时，发现 mode 都变了.目前没有太好的变法，只能每次 checkout .，就能还原 git 工程下的文件权限
