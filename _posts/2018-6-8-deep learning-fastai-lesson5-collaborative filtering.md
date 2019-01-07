---
layout: post
title:
modified:
categories: Tech
 
tags: [fastai,deep-learning]

  
comments: true
---


<!-- TOC -->

- [梯度计算的一些基本方法](#梯度计算的一些基本方法)
- [embedding](#embedding)
    - [什么是embedding](#什么是embedding)

<!-- /TOC -->


### 梯度计算的一些基本方法

已经知道的:

* Full-batch GD(The original ever)
* SGD (Choose a single random sample to cal at a time)
* Min-batch SGD 
* Learning rate finding way (fastai)
* SGD with restart (fastaai)
* different layer with different learning rate (fastaai)

来点更多的:

[Jacobian 和Hassian矩阵](http://jacoxu.com/jacobian%E7%9F%A9%E9%98%B5%E5%92%8Chessian%E7%9F%A9%E9%98%B5/)

[梯度下降法综述](https://blog.csdn.net/heyongluoyao8/article/details/52478715)

[Origin Version](https://arxiv.org/pdf/1609.04747.pdf?)

[Momentum SGD 动量梯度下降法](https://blog.csdn.net/yinruiyang94/article/details/77944338)

[Momentum SGB better](https://towardsdatascience.com/stochastic-gradient-descent-with-momentum-a84097641a5d)

[weigth decay](https://blog.csdn.net/amds123/article/details/69621688)

weight decay就是ml中正则化的正则项系数，在nn的梯度下降法里，自然也是可以存在的

[batch normalazation](https://blog.csdn.net/whitesilence/article/details/75667002)


### embedding

都包含了embedding的思想，即电影由factors个特征表征，用户对电影的喜爱也用factors个特征表征(这里为了计算方便，个数相同)

Ng里:

X为电影特征矩阵(n_moives x factors)
Y为用户特征矩阵(n_users x factors)

X*Y` (矩阵乘法) 就可以得到所有预测值. X和Y是通过线性回归方式估计的

```
n_movies x factors x factors x n_users  =  n_movies x n_users
```

1. 直接实现:

 movie embedding dotmultipy  user embedding then add  bias,然后sigmoild,直接用momentum SGD.

 又叫shadow learning , without hidden layer?

```python
def get_emb(ni,nf):
    e = nn.Embedding(ni, nf)
    e.weight.data.uniform_(-0.01,0.01)
    return e

class EmbeddingDotBias(nn.Module):
    def __init__(self, n_users, n_movies):
        super().__init__()
        (self.u, self.m, self.ub, self.mb) = [get_emb(*o) for o in [
            (n_users, n_factors), (n_movies, n_factors), (n_users,1), (n_movies,1)
        ]]
        
    def forward(self, cats, conts):
        users,movies = cats[:,0],cats[:,1]
        um = (self.u(users)* self.m(movies)).sum(1)
        #注意 两个变量的bias都是直接相加了
        res = um + self.ub(users).squeeze() + self.mb(movies).squeeze()
        #这个是额外的处理，讲结果sigmoid [0-1]化， 再重新计算real rating 
        res = F.sigmoid(res) * (max_rating-min_rating) + min_rating
        return res.view(-1, 1)

```

2. 加了hidden/output layer的网络:

move embedding + user embedding as nn input layer

a hidden layer (drop  + Relu)  size: 2*factors-->10

a output layer  size :  10-->1 (drop + sigmoid(不是必需的))

```python
class EmbeddingNet(nn.Module):
    def __init__(self, n_users, n_movies, nh=10, p1=0.05, p2=0.5):
        super().__init__()
        (self.u, self.m) = [get_emb(*o) for o in [
            (n_users, n_factors), (n_movies, n_factors)]]
        self.lin1 = nn.Linear(n_factors*2, nh)
        self.lin2 = nn.Linear(nh, 1)
        self.drop1 = nn.Dropout(p1)
        self.drop2 = nn.Dropout(p2)
        
    def forward(self, cats, conts):
        users,movies = cats[:,0],cats[:,1]
        x = self.drop1(torch.cat([self.u(users),self.m(movies)], dim=1))
        x = self.drop2(F.relu(self.lin1(x)))
        return F.sigmoid(self.lin2(x)) * (max_rating-min_rating+1) + min_rating-0.5
```

#### 什么是embedding

[Why You Need to Start Using Embedding Layers](https://towardsdatascience.com/deep-learning-4-embedding-layers-f9a02d55ac12)


```
Turns positive integers (indexes) into dense vectors of fixed size.
```

是不是可以理解为index one-hot coding(m维)后，再加一层full layer的转换(nxm),embedding的维度为n维?即：

```
1 --> 1xm  --> 1xm x m x n (n < m) --> 1 x n 
```

```
 Because the embedded vectors also get updated during the training process of the deep neural network, we can explore what words are similar to each other in a multi-dimensional space. By using dimensionality reduction techniques like t-SNE these similarities can be visualized.
```





