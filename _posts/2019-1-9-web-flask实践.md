---
layout: post
title:
modified:
categories: Tech

tags: [python, web]

comments: true
---

<!-- TOC -->

- [工程架构](#工程架构)
  - [import 相对路径的问题](#import-相对路径的问题)
  - [config](#config)
- [flask-sqlchemy 问题](#flask-sqlchemy-问题)
  - [db checker](#db-checker)
  - [hooks](#hooks)

<!-- /TOC -->

## 工程架构

### import 相对路径的问题

只能 `sys.path.append('.')`?感觉很傻.
相对路径:

```py
import . from xxx
import .. from xxx
```

注意:

```sh
1. 顶层模块不要用，子目录(package)引用自身，或者跨目录，可以考虑 .和..
2. __init__.py 里,把本模块全部import一遍
```

使用两种方式都 ok 了

### config

在 config 文件里配置的选项，都可以以字典方式取出来:

```py
app.logger.info('After request %s ' % app.config['DATABASE_QUERY_TIMEOUT'] )
```

## flask-sqlchemy 问题

首先要 create database.

下了个 mysql-client

```sql
CREATE DATABASE FP;
USE FP;
```

然后在 config.py 里指定 database 时才正常:

```py
SQLALCHEMY_DATABASE_URI="mysql+pymysql://root:root@localhost:3306/FP"
```

然后为了正确建立 table,必须:

```sh
# before create_all, import all models
# 顺序都不能有问题,
from .database import  db
from ..models import users
app = create_app()
bootstrap = Bootstrap(app)
app.logger.info('init db!')
db.init_app(app)
```

### db checker

基于`get_debug_queries`日志分流.db 的归 db ,逻辑的归逻辑. 更好的定位各种问题

### hooks

除了上面的 signal 机制，还有 flask 通过 decorator 装饰出来的 hook 在各种时机调用.
