---
layout: post
title:
modified:
categories: Tech
tags: [thinksns]
comments: true
---


## 先看懂router

如访问动态:
```sh
http://localhost:8080/spa/#/feeds?type=new
http://localhost:8080/spa/#/feeds?type=hot
```
访问资讯:
```sh
http://localhost:8080/spa/#/news

```
Q: 前端处理router? 印象都是后端做的,但要注意这是spa应用.
Q: 为何url有个#号
router将router url与对应的vue组件绑定起来.

* routes.js
定义各个router模块固定， as routes
也就是明确什么url访问什么组件.
* index.js
基于上面的routes对象，配置Vue-router模块,导出的router，用来配置项目里的Vue实例(main.js)


