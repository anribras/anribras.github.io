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
    - [bash优化](#bash优化)
    - [Xlaunch](#xlaunch)
    - [配色](#配色)
    - [wsl权限问题](#wsl权限问题)

<!-- /TOC -->

新购置了Sangsum 860evo 512G ssd,打算替换原笔记本机械盘.

机器: Thinkpad X260, 单机械硬盘 512G, 已安装win10+ubuntu16.04.

保底目标: 无缝迁移原hdd到ssd上,特别ubuntu16.04作为工作环境,不能有影响.

更精致的目标: windows重装, ubuntu个人部分到ssd, 工作部分到外接的hdd(当移动硬盘用咯),这样插上盒子就是工作环境, 拔掉盒子就是自己的个人环境.

如果hdd外接硬盘盒子能直接使用,那就可在新ssd上随意折腾...只要不影响原工作.



### 安装系统

1. 启动盘准备. 
用refus制作win10 ios镜像即可

2. ssd分区 for win10 
分区助手, 4k对齐.

系统盘60G. Work盘150G.


3. 拆换硬盘,用win10启动盘安装win10. 

4. 原机械盘外接到硬盘盒子; ubuntu启动盘也插上;

机械盘外接能使用进原ubuntu.完美.不用重新装ubuntu了.

至此，新win10运行在ssd上，原工作环境运行在外接的hhd硬盘盒子.


### win10

基于`1803 iso`,其他软件该装的都装,注意有个叫`神龙`的系统激活软件最好用.

#### wsl

以前就听过，理论上的完美双系统, 必须试验下.

en...体验各方面介于pure linux 和 virtual machine之间.

#### bash优化

基于叫`wsl-open`的折腾，wsl自带的bash太丑了.

tmux+vim的环境怎能错过?按自己的以前的config配置一遍... 难点自然是`vim youcompleteme`了.

tmux原来的tmux.conf不能直接用，一行一行实验最后写了个新的.

遇到一个大坑是从bash到windows的clipboard问题，网上有牛人来解决:
[这里](https://github.com/Microsoft/WSL/issues/892),总体是下载一个`XLaunch`的东西,另外vim需要编译`+clipboard`的支持,还是挺麻烦的

#### Xlaunch

不装不知道，这东西非常牛，直接将x11的界面弄到win10上了，使得在wsl开发GUI也是可能，比如我正在做的这个cross-platform displaySim.


#### 配色

这个折腾我太久了，配色是个大问题.找了半天发现codesschool还可以，再修改下Green bold的配色，文件夹也看的清楚了:
```
mintty theme:base16-codeschool.minttyrc
oh-my-zsh: risto 
```

先这样吧,折腾的时间耗不起:

markdown自然还是用vscode来写，原来的七牛截图不能用，把原来的七牛配置稍微改下就好:
![2018-08-20-15-15-53](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-08-20-15-15-53.png)

working bash最终效果:

![2018-08-22-14-51-30](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-08-22-14-51-30.png)

#### wsl权限问题

美好的故事是把文件都房子放在window盘，wsl mount在了/mnt/x/下,尽情用之...但发现所有文件的权限都成了777.

比如原linux git仓库的文件在在windows/project下，再用wsl去读时，全是modified的状态...

wsl固有的文件系统方式，解释[在这里](https://github.com/Microsoft/WSL/issues/2182)

这也是1坑，先记在这里.

这个貌似是[解决方案](https://blogs.msdn.microsoft.com/commandline/2018/01/12/chmod-chown-wsl-improvements/)

```
sudo umount /mnt/c
sudo mount -t drvfs C: /mnt/c -o metadata,uid=1000,gid=1000,umask=22,fmask=111
```

不试不知道...原来之前的文件夹被加背景色，就是因为权限加成了777! 改了之后，发现mintty theme都能看了.没必要去各种尝试了..

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

需要注意在wsl下， 文件权限都是755, 特别是看git 在windows下的文件时，发现mode都变了.目前没有太好的变法，只能每次checkout .，就能还原git 工程下的文件权限











