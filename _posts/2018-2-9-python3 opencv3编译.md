---
layout: post
title:
modified:
categories: Tech
tags: [python,opencv]

  
comments: true
---
<!-- TOC -->

- [步骤](#步骤)
- [编译错误](#编译错误)

<!-- /TOC -->

[参考](http://blog.csdn.net/keith_bb/article/details/65447707?locationNum=6&fps=1)


### 步骤
```
sudo apt-get install libgtk2.0-dev pkg-config libavcodec-dev libavformat-dev libswscale-dev libtbb2 libtbb-dev libjpeg-dev libpng12-dev libtiff4-dev libjasper-dev libdc1394-22-dev

cd  opencv-3.4.0
mkdir build_python
cd build_python
touch configure_python_x64.sh
./configure_python_x64.sh
make 
sudo make install
```
configure内容如下:
```
#!/bin/sh
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr/local \
        -DBUILD_TIFF=ON \
        PYTHON3_EXECUTABLE=/usr/bin/python3 \
        PYTHON_INCLUDE_DIR=/usr/include/python3.5 \
        PYTHON_INCLUDE_DIR2=/usr/include/x86_64-linux-gnu/python3.5m  \
        PYTHON_LIBRARY=/usr/lib/x86_64-linux-gnu/libpython3.5m.so \
        PYTHON3_NUMPY_INCLUDE_DIRS=/usr/lib/python3/dist-packages/numpy/core/include/ \
        ..
```



### 编译错误

1. `grfmt_png.cpp:387:82: error: ‘Z_FIXED’ was not declared in this scope`

在grfmt_png.cpp中添加`#define Z_FIXED 4`。

2. `grfmt_tiff.cpp:132:12: error: 'tmsize_t' does not name a type`

[解决方案](http://answers.opencv.org/question/178962/compile-time-errors-with-cvtiffdecoderbufhelper/)

在cmake时添加`-DBUILD_TIFF=ON`。


3. 遇到一个贼诡异的问题，`pip3 -v` 不能用了,不确定是由这引起

[解决方案](https://stackoverflow.com/questions/47955397/pip3-error-namespacepath-object-has-no-attribute-sort)

```
```


