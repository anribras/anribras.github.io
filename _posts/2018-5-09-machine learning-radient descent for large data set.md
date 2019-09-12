---
layout: post
title:
modified:
categories: Tech

tags: [machine-learning]

comments: true
---

<!-- TOC -->

- [批量梯度下降](#批量梯度下降)
- [随机梯度下降](#随机梯度下降)
- [mini 批量下降](#mini批量下降)
- [如何确保收敛和确定学习速率](#如何确保收敛和确定学习速率)
- [online learning](#online-learning)
- [mapreduce 和并行计算](#mapreduce-和并行计算)

<!-- /TOC -->

### 批量梯度下降

Batch gradient descent,就是最普通的梯度下降法，`batch`的理解是对每个参数,需要综合所有 m 个样本来`批量`计算.

### 随机梯度下降

SGD, stochastic gradient descent
随机梯度就是`随机取样`一些训练数据，替代整个训练集，在其上作 target function 的梯度下降,每次迭代只用使用`１个样本`.

大尺度数据时，可节省计算，但是可能收敛到`局部最优`.

### mini 批量下降

选取的样本数 b 介于 1 和 m 之间，(b=10-100,typically)也就是 mini-batch 的含义.

![2018-05-10-09-46-21](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-05-10-09-46-21.png)

并且使得`并行计算`有可能,从 m 中同时计算若干个 b 的路径，最后看哪个路径效果最好。

### 如何确保收敛和确定学习速率

BGD 就是检查$J(\Theta)$与迭代次数 n 的关系.

SGD，不用检查所有的样本，比如每 1000 次迭代时，看前面 1000 个样本的`样本代价函数`$J(\Theta,mean(x^{i->i+1000},y^{i->i+1000}))$
平均值即可。

震荡下降，更小的学习速率有更小的震荡幅度，收敛的更慢(图１红线),

1000 这个 batch observe point 越大，曲线越平滑.(图 2)

很大震荡，可能是 batch observe point 太小，如果增大后，曲线收敛仍然 ok。(图 3)

observe point 也比较大，但是仍然发散，考虑减少学习速率，也可能是算法有问题(图 4)

![2018-05-10-10-08-43](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-05-10-10-08-43.png)

学习速率选择:

1.直接选择一个合理的常数;

2.随着越接近最小值，(n 增加)，希望震荡减少.

$$\alpha = \frac{const1}{const1+iteration\ n}$$

### online learning

应该就是基于 streaming 的在线数据流学习技术了.

on line 体现在,样本不是一开始固定的，而是`来一个处理一个`.当然实际可能是`来一大批，就处理一大批`.

对于刚获得的用户数据(样本 x,y=0/1 已知),学习一次参数，感觉就是前面的 SGD or MBGD.

来对哪些未做选择(y 未知),但是留下了特征(X,网上记录，选择了某些选项)的用户,(预测其可能选择我们服务的概率，从而估计利润，调整服务内容...)

online learning 可达到的一个效果:会根据实际用户的情况，模型自己调适。

- CTR 问题模型(点击率估计)

![2018-05-10-10-42-33](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-05-10-10-42-33.png)

每个手机都有在 X 下被点击的概率，很适合 mutivariale logistic regression 模型.

对于手机,是否被点击(y),一个可能的手机特征(X)如下:

价格、屏幕尺寸，摄像头分辨率、外观评分，网络热度，用户是否浏览过,用户搜索次数...

而后面两个特征,就是 online 时 可及时获得的,由此得到的样本进行训练.

### mapreduce 和并行计算

梯度下降法的一个 map-reduce 模型,这是很显然的

![2018-05-10-11-26-20](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-05-10-11-26-20.png)
