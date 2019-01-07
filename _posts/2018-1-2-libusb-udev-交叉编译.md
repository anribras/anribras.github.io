---
layout: post
title:
modified:
categories: Tech
 
tags: [libusb]

  
comments: true
---
<!-- TOC -->

- [前言](#前言)
- [libudev](#libudev)
- [移植过程](#移植过程)
- [一个大坑](#一个大坑)

<!-- /TOC -->

### 前言

交叉编译的版本是libusb的[最新版本1.0.21](https://sourceforge.net/projects/libusb/files/)。

libusb-1.0.16以上因支持hotplog依赖udev,编译期间遇到了麻烦即libudev。

网上搜后并没有解决方案，倒是很多configure时`--enable-udev=no`,而我是需要这个功能的，只好自己来了。

### libudev

libudev并没有发布源码包，官方的udev是实现在`systemd`下的。
只好从debian源里找，发现还真有已经编译好的arm库，选择`armel`,下载。

关于什么是`armel/armhf`，可以参考[这篇文章](http://www.veryarm.com/872.html)。

下面的链接download下来解包:
```
https://packages.debian.org/wheezy-backports/armel/libudev1/download
```

解dev包的方法:
```
法１:
ar -vx xx.deb 
tar -xzvf data.tar.gz  
法2:
dpkg -x xx.dev /xx/xx
```

### 移植过程

交叉编译configure时，1个习惯是在新dir,而不是直接configure，好处是编译更多平台而互不干扰。
```
mkdir build-sp
cd build-sp
touch configure-sp.sh
```

另外1个习惯是将编译依赖和编译后的库用相同目录存放，(用`--prefix`指定，同时`CFLAG`和`LDFLAG`也指定为同一目录),当移植时，直接复制所有库即可。

configure脚本如下:
```
#!/bin/sh

TOOLCHAIN_ROOT=/home/litecar/toolchain/fsl-linaro-toolchain
INSTALL_PREFIX=/home/install-prefix/sunplus

../configure \
--prefix=$INSTALL_PREFIX \
--host=arm-linux \
--enable-static=yes \
--enable-shared=no \
CC="$TOOLCHAIN_ROOT/bin/arm-none-linux-gnueabi-gcc -fPIC -march=armv7-a -marm -mthumb-interwork"  \
CFLAGS="-I$INSTALL_PREFIX/include" \
LDFLAGS="-L$INSTALL_PREFIX/lib" 
#CPPFLAGS="-I$INSTALL_PREFIX/include" \
```

`/home/install-prefix/sp`下的库组织结构如下，这样libusb依据上面的脚本可以找到libudev的依赖路径

```
├── include
│   └── libudev.h
├── lib
│   ├── libudev.so -> libudev.so.1.3.5
│   └── libudev.so.1.3.5
```
### 一个大坑

在运行configure libusb时，提示错误找不到libudev.查看configure文件可以知道其检查依赖库的原理，就是用cross-compile gcc去编译依赖库的最小代码，以检查依赖库是否工作。configure报错并不提示为何依赖库不工作。　

于是自己仿照configure编写简单的代码:
```
int udev_new();

int main(int argc, char* argv[])
{
	return udev_new();
}

```
并直接gcc编译:
```
/home/litecar/toolchain/fsl-linaro-toolchain/bin/arm-none-linux-gnueabi-gcc -fPIC -march=armv7-a -marm -mthumb-interwork -mfloat-abi=hard -mfpu=neon main.c -o main -ludev -lc -L/home/install-prefix/sunplus/lib

```
终于找到原因是`该版本的libudev1对应的libc>=2.17`,而toolchain依赖的libc版本才2.13。

重新编译glic不太科学，于是回退找libudev的更低版本，还是发现有满足`glibc>=2.13`的。

重下载编译之，done。
```
./configure_sp.sh
make
make install
```
