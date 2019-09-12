---
layout: post
title:
modified:
categories: Tech

tags: [deep-learning]

comments: true
---

<!-- TOC -->

- [流程](#流程)

<!-- /TOC -->

## 流程

流程如下:

```shell
conda install jupyter

mkdir /opt/jupyternote/root

#输入密码后，保存sha1:xxx的字符串;
python -c "import IPython;print(IPython.lib.passwd())"

#生成默认配置文件
jupyter notebook --generate-config --allow-root

# 编辑并添加内容:

c.NotebookApp.ip = '*'
c.NotebookApp.allow_root = True
c.NotebookApp.open_browser=False
c.NotebookApp.port = 8888
c.NotebookApp.password = u'sha1:xxx'
c.ContentManager.root_dir = '/opt/jupynote/root'

#启动
nohup jupyter notebook > ju.log 2>&1 &

```

github 直接支持 jupyter 啊.简直不能太方便.

顺便安装个 jupyer 的扩展插件.

```shell
conda install -c conda-forge jupyter_contrib_nbextensions
```

用 jupyter 写笔记，真的无敌，太喜欢了
