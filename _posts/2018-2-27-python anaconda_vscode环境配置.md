---
layout: post
title:
modified:
categories: Tech
tags: [tools, python]

comments: true
---

<!-- TOC -->

- [最新 anaconda 下载和安装](#最新-anaconda-下载和安装)
- [anaconda 使用](#anaconda-使用)
- [opencv 安装](#opencv-安装)

<!-- /TOC -->

[参考文章](http://blog.csdn.net/songyingxu/article/details/78940305)

### 最新 anaconda 下载和安装

[anaconda 下载](https://repo.continuum.io/archive/)

unbuntu 16.04 选择`Anaconda3-5.1.0-Linux-x86_64.sh`

安装:

```
chmod  755 Anaconda3-5.1.0-Linux-x86_64.sh
./Anaconda3-5.1.0-Linux-x86_64.sh
```

一路提示，把/opt/anaconda/bin export 到 PATH 中。
此时运行 python 已经是 anaconda 里的默认 python3.6 而不是本机的 python3.5 了

Vscode 已经就绪，需要增加设置到 setting 就可以运行 anonconda 环境下的 python

```
    "python.pythonPath": "/opt/anaconda3/bin/python",
    "python.autoComplete.extraPaths": [
        "/opt/anoconda3/lib/python3.6/dist-packages",
    ],
```

### anaconda 使用

创建新虚拟环境,可以不同版本和项目间隔离

```
conda create -n python35 python=3.5
```

展示新的环境

```
conda info -e
```

使用新环境

```
source activate python35
source deactivate python35
```

删除环境

```
conda remove -n python35 --all
```

设置国内镜像

```
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --set show_channel_urls yes
```

安装

```
conda install xxx
pip install xx
```

### opencv 安装

[手动下 whl 文件](https://pypi.python.org/pypi/opencv-python)

conda install opencv 只有 3.2.0 的版本，用 pip 的比较新为 3.4.0:
带 contrib 库的用后面一个即可

```
pip install opencv-python
pip install opencv-contrib-python
```
