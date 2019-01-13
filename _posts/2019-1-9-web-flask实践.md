---
layout: post
title:
modified:
categories: Tech
 
tags: [python,web]

comments: true
---

<!-- TOC -->

- [工程架构](#工程架构)
    - [import 相对路径的问题](#import-相对路径的问题)
    - [config](#config)
- [flask-sqlchemy问题](#flask-sqlchemy问题)
    - [db checker](#db-checker)
    - [hooks](#hooks)

<!-- /TOC -->


## 工程架构

### import 相对路径的问题
只能 `sys.path.append('.')`?感觉很傻.
相对路径:
```
import . from xxx
import .. from xxx
```
注意:
```
1. 顶层模块不要用，子目录(package)引用自身，或者跨目录，可以考虑 .和..
2. __init__.py 里,把本模块全部import一遍
```
使用两种方式都ok了

### config

在config文件里配置的选项，都可以以字典方式取出来:
```py
app.logger.info('After request %s ' % app.config['DATABASE_QUERY_TIMEOUT'] )
```

## flask-sqlchemy问题

首先要create database.

下了个mysql-client
```sql
CREATE DATABASE FP;
USE FP;
```
然后在config.py里指定database时才正常:
```py
SQLALCHEMY_DATABASE_URI="mysql+pymysql://root:root@localhost:3306/FP"
```

然后为了正确建立table,必须:
```
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

基于`get_debug_queries`日志分流.db的归db ,逻辑的归逻辑. 更好的定位各种问题


### hooks

除了上面的signal机制，还有flask通过decorator装饰出来的hook在各种时机调用.


