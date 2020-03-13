---
layout: post
title:
modified:
categories: Tech
tags: [jetbrains]
comments: true
---

clion只支持cmake.

当然cmake语法是很重要的了.

直接写CMakeLists.txt.

但用clion导入原有makefile工程是有技巧的,`不然没法补全,没法build`.

```sh
File->
New CMake project from sources->
Import as CMake project->
Import as new Cmake project
```

会生成导入了各种.cpp/.h的基础CMakeLists.txt,并且可以直接在ide build/debug等

然后基于CMakeLists.txt做自己的修改,比如so库,生成bin等等.
