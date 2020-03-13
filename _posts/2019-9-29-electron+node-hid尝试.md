---
layout: post
title:
modified:
categories: Tech
tags: [electron,usb]
comments: true
---



electron不能再wsl下跑，还是只能windows下安装node.js

## electron 安装

就按文档来好了，分发打包用`electron-forge`,记得都全局安装.yarn安装不上就换npm就好

## electron-vue

这个算是最佳工程实践了

但是依赖vue-cli 2,安装个全局的.

<https://github.com/SimulatedGREG/electron-vue>

## node-hid

`node-hid`是node.js访问usb的包，肯定依赖libusb这样的底层c/c++库，node下编译工具是node-gyp.

这货又依赖各种windows-build-tool: 看node-gyp文档:<https://github.com/nodejs/node-gyp>

期间还需要用python2,于是用conda创建环境，遇到了个问题无法`activate`,输入命令:

```sh
conda init powershell
```

```sh
conda create python=python2.7 name=py27
conda activate py27
```

## usb

上面那个只能针对hid设备.

node.js封装的libusb才是最好用的
```sh
yarn add usb
```
