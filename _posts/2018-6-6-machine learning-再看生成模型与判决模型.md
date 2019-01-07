---
layout: post
title:
modified:
categories: Tech
 
tags: [machine-learning]

  
comments: true
---

<!-- TOC -->

- [ref](#ref)
- [判决模型](#判决模型)
- [生成模型](#生成模型)
- [参数估计](#参数估计)
    - [maximum likelihood estimation (MLE)](#maximum-likelihood-estimation-mle)
    - [maximum a posteriori estimation (MAP)](#maximum-a-posteriori-estimation-map)

<!-- /TOC -->

### ref

[生成模型与判别模型](https://blog.csdn.net/zouxy09/article/details/8195017)

[cs229-notes](http://cs229.stanford.edu/notes/cs229-notes2.pdf)

[difference between generative and discriminative model](https://stackoverflow.com/questions/879432/what-is-the-difference-between-a-generative-and-discriminative-algorithm)


### 判决模型

所谓`判决`,即:给我一个样本集合,通过训练,我得到判决函数y=F(x),再来新的x的时候,根据F(x)直接可以预测y; 

或者我估计一个概率模型,即估计$$P(y|x)$$
的概率分布(即各种具体的x条件下,出现各种y的概率),有了个概率分布,自然也能计算新的x条件下,各种y的概率,取其中最大的就是在预测y的过程,即

$$f(x)=arg\  \underset{y}{max} P(y|x)$$

如何求判决函数?如何求模型$$P(y|x)$$
?正是supervisor learning各种算法解决的问题.


### 生成模型

生成模型,是直接估计$$P(x,y)$$
即联合概率模型.有了它,再结合beyes公式,也可以求得$$P(y|x)$$
,假设y=j:

$$P(y=j|x)=\frac{P(x,y=j)}{P(x)}=\frac{P(y=j)P(x|y=j)}{\sum_{i}P(y_{i})*P(x|y_{i})}$$

和判决模型得到同样的结果.不过上面的公式是先估计$$P(x|y)和P(x)$$
对于$$P(x|y)$$
,可以理解某个标签(y)对应的特征分布(x).

ng里的例子:生成模型先求大象(y=0)的特征分布,马(y=1)的特征分布,然后给出1个未知动物的特征(x),看其最符合哪个分布,就是什么动物(predict y).

而判决模型更直接: 根据现有训练集合,训练一个模型,之后给我特征,直接告诉你是什么动物.


### 参数估计

参数估计是统计意义.某个事情已经发生,去估计其发生的概率分布,或求解一个模型.模型可以用一些参数来描述,即$$f(x;\theta)$$

如何做参数估计,要了解`频率学派`和`贝叶斯学派`.
[频率学派和贝叶斯学派的参数估计](https://blog.csdn.net/wzgbm/article/details/51721143)

#### maximum likelihood estimation (MLE)
对于统计模型来说,比较常见的方法有`极大似然估计`.假定参数是一个变量,并认为已经发生的就是概率最大的事情.即似然函数最大, 通过求似然函数极值,求解参数.

[似然函数的理解](https://www.zhihu.com/search?type=content&q=%E4%BC%BC%E7%84%B6%E5%87%BD%E6%95%B0)

然而MLE只能找到全局最优解,有的情况下,更是没法显示求得似然函数.

#### maximum a posteriori estimation (MAP) 

从贝叶斯学派的观点,MLE只是MAP的特殊情况,即先验概率为均匀分布的情况,贝叶斯学派认为先验概率也很重要.这也是生成模型里用到的思想.

[wiki](https://en.wikipedia.org/wiki/Maximum_likelihood_estimation)讲的也很清楚







