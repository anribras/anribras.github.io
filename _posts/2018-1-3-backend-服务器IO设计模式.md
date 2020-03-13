---
layout: post
title:
modified:
categories: Tech
tags: [backend]
comments: true
---

<!-- TOC -->

- [４种IO方法](#４种IO方法)
- [reactor和proactor](#reactor和proactor)
  - [基本概念](#基本概念)
  - [reactor和IO多路复用的区别](#reactor和IO多路复用的区别)
  - [proactor](#proactor)

<!-- /TOC -->

### ４种IO方法

[阻塞，非阻塞，多路IO复用，async调用](http://blog.csdn.net/historyasamirror/article/details/5778378)

### reactor和proactor

[Comparing Two High-Performance I/O Design Patterns](http://www.artima.com/articles/io_design_patternsP.html)

#### 基本概念

<https://www.cnblogs.com/me115/p/4452801.html>
<http://blog.csdn.net/roger_77/article/details/1555170>
<http://blog.csdn.net/russell_tao/article/details/17452997>

#### reactor和IO多路复用的区别

前者是一个完整的设计模式,后者仅仅是一种io方法。

reactor:

- 封装了IO复用的使用
- 响应分类　读的，写的，超时的..
- 事件分发和应用处理逻辑分离，应用注册hander,处理hander即可，不用关心怎么来的。
- 如果Reactor多任务并发，是在一个线程中进行的,为单线程模式，一般为这种模式
- 异步回调产生后，还需要才callback里处理IO

#### proactor

ASYCIO 内核完成了IO动作才回调，callback里是取IO后的结果
