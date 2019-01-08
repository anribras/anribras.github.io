---
layout: post
title:
modified:
categories: Tech
tags: [python,web]

  
comments: true
---
<!-- TOC -->


<!-- /TOC -->


可以通过配置文件方式启动gunicorn:
```
gunicorn -w 4 -c config.py main.py
```
config.py里通过下面的函数注册钩子函数:

```py
import threading
from ota_dir_monitor import dir_monitor

def on_starting(server):
    server.log.info("dir monitor")
    t = threading.Thread(target=dir_monitor,name='dir_monitor')
    t.start()

def on_reload(server):
    server.log.info("dir monitor")
    t = threading.Thread(target=dir_monitor,name='dir_monitor')
    t.start()
```
就是想在hook里启动一个监控文件夹的线程,但是遇到了找不到模块的错误:
```
ModuleNotFoundError: No module named 'ota_dir_monitor'
```

google了一下，可能和文件组织有关。



