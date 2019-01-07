---
layout: post
title:
modified:
categories: Tech
 
tags: [deep-learning]

  
comments: true
---
<!-- TOC -->

- [注意](#注意)
- [神经网络模型的直观展示](#神经网络模型的直观展示)
- [为何训练deep nerual networks很困难？](#为何训练deep-nerual-networks很困难)
    - [梯度消失和爆炸](#梯度消失和爆炸)
    - [初始化weight的选择](#初始化weight的选择)
    - [sigmoid as output layer](#sigmoid-as-output-layer)
    - [选择不同的gradient descent的困难](#选择不同的gradient-descent的困难)
    - [网络架构，多少层，什么层?](#网络架构多少层什么层)
    - [超参数如何调优](#超参数如何调优)

<!-- /TOC -->

### 注意

1. 注意

```
Summing up, a more precise statement of the universality theorem is that neural networks with a single hidden layer can be used to approximate any continuous function to any desired precision. In this chapter we'll actually prove a slightly weaker version of this result, using two hidden layers instead of one.
```

### 神经网络模型的直观展示

[立体展示2 hidden layer 神经网络模型的工作原理](http://neuralnetworksanddeeplearning.com/chap4.html)

### 为何训练deep nerual networks很困难？ 

#### 梯度消失和爆炸

反向传播时，随着layer越往前，梯度也是变的越来越小。the gradient tends to get smaller as we move backward through the hidden layers.

反之则是梯度爆炸

![2018-06-27-16-08-45](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-06-27-16-08-45.png)

简单的表达:

![2018-06-27-16-19-49](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-06-27-16-19-49.png)

复杂的网络里(bp算法里的推导):

![2018-06-27-16-39-42](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-06-27-16-39-42.png)

总之，这两类问题的根源是，It's that the gradient in early layers is the product of terms from all the later layers.When there are many layers, that's an intrinsically unstable situation.

前面的layer的梯度是后面layer梯度的乘积，后面layer的梯度如果变小，那前面乘积起来自然也变小，后面layer的梯度在增加(超过1的乘积)，那么前面的梯度也会增大


当前的业界的解决:Relu?

#### 初始化weight的选择

#### sigmoid as output layer 

前面学习bp算法时，sigmoid as ouputlayer ，算梯度时，由与sigmoild本身的特性，求导后，接近0 or 1时,函数值变化变的很小，即参数学习的速率大大降低，这是另外一种`梯度消失`.

当前的业界的解决:softmax + log loss? or sigmoid output + cross-entry?

#### 选择不同的gradient descent的困难

什么情况适应什么样的momentum with stochastic gradient descent?

解决：fastai里的那些技巧?

#### 网络架构，多少层，什么层? 

#### 超参数如何调优
