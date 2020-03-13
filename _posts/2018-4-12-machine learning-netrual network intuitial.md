---
layout: post
title:
modified:
categories: Tech
tags: [machine-learning]

comments: true
---

<!-- TOC -->

- [Nenrual network](#nenrual-network)
- [预测函数](#预测函数)
- [算法求解](#算法求解)

<!-- /TOC -->

### Nenrual network

下面简称神经网络为 nn 好了.

印象较深的点:

- nn 的数学表达.

矩阵，向量的表达最重要，这个图需要时刻放在脑海里:
![2018-04-13-12-57-32](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-04-13-12-57-32.png)

前向传播，输入层，隐藏层，输出层

- nn 自己学习特征

- sigmoid 函数解决线性不可分的逻辑与或，异或问题

貌似只是 logistic regression 本身的特性，和 nn 本身并没有关系,只是用在了神经元的激励函数上.

- 每个的第一个神经元为 bias
   一般每层的第 1 个神经元为 bias feature(x0=bias),额外添加的，比如 sigmoid 函数的特性来完成逻辑与或的计算。

- 逻辑运算

像异或这种非线性逻辑可以通过小型的 layer 2 网络来实现

### 预测函数

按 nn 的网络来,假设 3 层，每层的激励函数为${g(x)}$:

$$h_{\theta}(x)=g(g(X\vartheta_{1}^{T}))\vartheta_{2}^{T}...$$

### 算法求解

![2018-04-16-07-53-05](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-04-16-07-53-05.png)

![2018-04-16-07-56-15](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-04-16-07-56-15.png)

仍然是需要计算代价函数和梯度

- 正向传播

- 反向传播
  就是求梯度的一个优化算法,正向+反向才是完整的算法。

- 梯度检查

可以判断算法梯度下降的正确性，但不直接使用，计算代价太大

原理也比较简单,用导数(偏导数)的近似，并取一个较小的$\varepsilon$，计算结果来比较算法计算得到的梯度值:

$$\dfrac{\partial}{\partial\Theta}J(\Theta) \approx \dfrac{J(\Theta + \epsilon) - J(\Theta - \epsilon)}{2\epsilon}$$

- 对称现象

输入到中间层，网络训练得到的`特征`是一样的，冗余的.用随机初始化打破可能出现的对称现象

- 工程技巧

>矩阵向量化与矩阵还原,fminun 这种高级优化算法要用
>
>梯度检查
>
>随机值初始化而不是全 0，加快收敛
>
>y=[0;1;0..]化 就是标注样本

自动驾驶，拿到司机的真实 driver 情况，左转标注[1;0;0;...],右转标注[0;1;0]..直接将样本训练出网络，拿到真实样本后，开始预测...

- 代价函数向量化
  ![2018-04-17-12-44-52](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-04-17-12-44-52.png)

week5 的作业还真是姿势大涨。原来训练一个 BP 神经网络也是可以轻松实现的
