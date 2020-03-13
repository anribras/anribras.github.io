---
layout: post
title:
modified:
categories: Tech
tags: [python, web]

comments: true
---

<!-- TOC -->

- [flask](#flask)
- [gunicorn](#gunicorn)
- [nginx](#nginx)
- [搬瓦工 Centos 6.8 部署](#搬瓦工-Centos-68-部署)

<!-- /TOC -->

用 anaconda 里的 conda 做代替了 python virtualenv。任性。。。因为 anaconda 包括 ml 的东西。

### flask

- 新建 python 3.6 环境

```
conda create -n python36 python=3.6
```

- 激活

```
source activate python3.6
```

- flask 等安装

```sh
pip install flask uwsgi gunicorn flask-httpauth pyinotify
```

- uwsgi

python 专用 web 服务器，并不太好用，先放弃

```
sudo apt-get install uwsgi-python-plugin uwsgi-python-plugin3
```

### gunicorn

命令行跑起来 这个简单，我喜欢。
先安装个

```
pip install gunicorn
```

跑起来，4 个进程

```
gunicorn -w 4 -b 127.0.0.1:8000 ota-demo:app
```

也可以通过 config.py,为了启动其他的 py 代码，config.py
可以增加`on_starting`和`on_reload`的 hook

```
gunicorn -c config.py ota-demo:app
```

例如我就想用 hook 来连接 mysql，以及启动文件夹的 monitor 线程。

实际中发现，config.py 里定义的 py import error，可能工程文件的组织方式还不对

```py
import threading
from ota_dir_monitor import dir_monitor

def on_starting(server):
    server.log.info("dir monitor")
    t = threading.Thread(target=dir_monitor,name='dir_monitor')
    t.start()
```

在 flask 框架下有专门的异步任务框架`Celery`，以后再研究。

### nginx

ubuntu 安装

```
sudo add-apt-repository ppa:nginx/stable
sudo apt-get install nginx
```

配置了 2 个主机，1 个文件服务器(800)，1 个就是 python 的 web service,(nginx 80 forward 8000)

配置文件如下:

```
access_log /var/log/nginx/error.log;
error_log /var/log/nginx/access.log;

server {
    listen      80;
    server_name localhost;
    charset     utf-8;
    client_max_body_size 75M;

   location / {
           proxy_pass http://127.0.0.1:8000; # 这里是指向 gunicorn host 的服务地址
           proxy_set_header Host $host;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   }
}

server {
    listen       800;
    charset utf-8;
    server_name  localhost;
        root    /opt/study/python3/flask;
    autoindex       on;
    autoindex_exact_size    off;
    autoindex_localtime     on;
}

```

添加到 conf 到 conf.d 下,最好用软链接

```
ln -s /xx/xx/self.conf  /etc/nginx/conf.d/
```

启动

```
/etc/init.d/nginx restart
/etc/init.d/nginx reload
```

### 搬瓦工 Centos 6.8 部署

- anaconda3
  直接 scp 安装文件安装就好
- nginx
  [nginx 参考这个](http://blog.csdn.net/Colton_Null/article/details/78513268)

安装后提示缺库

```
error while loading shared libraries: libpcre.so.1:
```

找一下:

```
ldconfig - p |grep  libpcre
```

在/lib64/下呢 做个软连接好了:

```
ln -s libpcre.so.0.0.1 libpcre.so.1
```

运行方式 需要 nohup

```
nohup /opt/anaconda3/envs/python36/bin/python  ota_main.py > flask.log 2>&1 &
```

输出只能定位到文件，看日志用

```
tailf flask.log
```
