---
layout: post
title:
modified:
categories: Tech

tags: [deep-learning]

comments: true
---

<!-- TOC -->

- [softmax](#softmax)
- [dropout](#dropout)
- [mini-batch](#mini-batch)
- [gradient vanish](#gradient-vanish)
- [Relu 激活函数](#relu激活函数)
- [Maxout](#maxout)
- [learning rate](#learning-rate)
- [Weight Decay](#weight-decay)
- [DNN](#dnn)

<!-- /TOC -->

理解可以顺这 ng 课程讲的`全连接网络`来.

#### softmax

ng 课程里，output layer 是一个多元 logistic 模型.这里提到了`softmax`来做 output layer 的 activate function.

[softmax 详解](https://zhuanlan.zhihu.com/p/25723112)

当损失函数是`交叉熵`时.softmax 由于求导梯度很方便

- 损失函数
  Linear regression 用损失函数是`mse`.

Linear regression 用的`log损失`.

SVM 可看成用的`hinger损失`.

决策树时，分裂特征的标准为`条件熵`,`gini`等

#### dropout

有点类似 ensemable 的思想，每次 dropout 实际上在训练不同的网络，把他们组合起来，可以减少过拟合的风险。

[dropout](https://blog.csdn.net/stdcoutzyx/article/details/49022443)

dropout 　 rate = p%,训练好后得到 w,在 test 时，w=w\*(1-p%).why?

![2018-05-14-16-25-53](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-05-14-16-25-53.png)

#### mini-batch

梯度下降时，前面 ng 的课讲过了，SGD,BGD,Min-batch GD 的区别，可以理解 Min-batch GD 可以 faster,但是 better?为什么会 better.

#### gradient vanish

过深的网络，容易出现`梯度消失`的问题,用更好的 activation function!

对应的还有`梯度爆炸`的问题.

最新的解决思路是`highway network`和`深度残差学习`.

#### Relu 激活函数

![2018-05-14-15-58-30](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-05-14-15-58-30.png)

#### Maxout

ReLU 是 maxout 的一个特例

#### learning rate

lr 太大，梯度跳跃,可能迭代不收敛
lr 太小，收敛太慢

随着迭代增加，慢慢减少 lr.

不同参数用不同的 lr.

- adagrad

根据梯度自适应调节学习率的方法

![2018-05-14-16-06-58](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-05-14-16-06-58.png)

还有很多调整方法,RMSprop,Adadelta,AdaSecant,Adam,Nadam...

- Adam

用`动量`的思想解决因为梯度太小而`陷入局部最优`的问题,根据该原理有`adam`算法

![2018-05-14-16-11-38](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-05-14-16-11-38.png)

#### Weight Decay

权重萎缩.有些特征对最后的分类并没有帮助，在梯度下降时，该特征对应的参数可加一个萎缩系数，慢慢让其作用权重越来越小

![2018-05-14-16-16-50](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-05-14-16-16-50.png)

#### DNN

和全连接 FNN 的结构类似，只是激活函数可能换成了 ReLU,output layer 换成 softmax 之类的
