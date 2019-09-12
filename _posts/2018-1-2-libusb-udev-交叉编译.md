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

交叉编译的版本是 libusb 的[最新版本 1.0.21](https://sourceforge.net/projects/libusb/files/)。

libusb-1.0.16 以上因支持 hotplog 依赖 udev,编译期间遇到了麻烦即 libudev。

网上搜后并没有解决方案，倒是很多 configure 时`--enable-udev=no`,而我是需要这个功能的，只好自己来了。

### libudev

libudev 并没有发布源码包，官方的 udev 是实现在`systemd`下的。
只好从 debian 源里找，发现还真有已经编译好的 arm 库，选择`armel`,下载。

关于什么是`armel/armhf`，可以参考[这篇文章](http://www.veryarm.com/872.html)。

下面的链接 download 下来解包:

```sh
https://packages.debian.org/wheezy-backports/armel/libudev1/download
```

解 dev 包的方法:

```
法１:
ar -vx xx.deb
tar -xzvf data.tar.gz
法2:
dpkg -x xx.dev /xx/xx
```

### 移植过程

交叉编译 configure 时，1 个习惯是在新 dir,而不是直接 configure，好处是编译更多平台而互不干扰。

```
mkdir build-sp
cd build-sp
touch configure-sp.sh
```

另外 1 个习惯是将编译依赖和编译后的库用相同目录存放，(用`--prefix`指定，同时`CFLAG`和`LDFLAG`也指定为同一目录),当移植时，直接复制所有库即可。

configure 脚本如下:

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

`/home/install-prefix/sp`下的库组织结构如下，这样 libusb 依据上面的脚本可以找到 libudev 的依赖路径

```
├── include
│   └── libudev.h
├── lib
│   ├── libudev.so -> libudev.so.1.3.5
│   └── libudev.so.1.3.5
```

### 一个大坑

在运行 configure libusb 时，提示错误找不到 libudev.查看 configure 文件可以知道其检查依赖库的原理，就是用 cross-compile gcc 去编译依赖库的最小代码，以检查依赖库是否工作。configure 报错并不提示为何依赖库不工作。

于是自己仿照 configure 编写简单的代码:

```
int udev_new();

int main(int argc, char* argv[])
{
	return udev_new();
}

```

并直接 gcc 编译:

```
/home/litecar/toolchain/fsl-linaro-toolchain/bin/arm-none-linux-gnueabi-gcc -fPIC -march=armv7-a -marm -mthumb-interwork -mfloat-abi=hard -mfpu=neon main.c -o main -ludev -lc -L/home/install-prefix/sunplus/lib

```

终于找到原因是`该版本的libudev1对应的libc>=2.17`,而 toolchain 依赖的 libc 版本才 2.13。

重新编译 glic 不太科学，于是回退找 libudev 的更低版本，还是发现有满足`glibc>=2.13`的。

重下载编译之，done。

```
./configure_sp.sh
make
make install
```
