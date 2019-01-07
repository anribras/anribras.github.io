---
layout: post
title:
modified:
categories: Tech
 
tags: [machine-learning]

  
comments: true
---


<!-- TOC -->

- [Boosting与Bagging的区别](#boosting与bagging的区别)
- [Boosting](#boosting)
    - [Adaboost](#adaboost)
        - [About weak learner](#about-weak-learner)
            - [分类问题](#分类问题)
            - [回归问题](#回归问题)
        - [前向分步加法模型](#前向分步加法模型)
- [Boosting Tree](#boosting-tree)
    - [Gradient boost](#gradient-boost)
        - [Gradient boost decision tree(GBDT)](#gradient-boost-decision-treegbdt)
        - [如何实现更好的GBDT](#如何实现更好的gbdt)
        - [关于GBDT算法的weak learner的training:](#关于gbdt算法的weak-learner的training)
        - [参考阅读](#参考阅读)

<!-- /TOC -->

这是之前做的一个[笔记](https://github.com/anribras/machine-learning/blob/master/note/decision_tree_and_ensemble_model.ipynb)

[很好的参考](https://machinelearningmastery.com/gentle-introduction-gradient-boosting-algorithm-machine-learning/)

## Boosting与Bagging的区别

ensemble包括boosting,bagging,stack等等.

![2018-06-21-14-01-01](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-06-21-14-01-01.png)

bagging属于有放回抽样，每个弱分类器的traning数据不同，对树方法，可能从全部特征里，随机选部分特征。得到n个模型后，用`weighted average, majority vote or normal average`这些方法最终ensemable一个模型,
降低variance,倒有点像nn里的`drop out`

常见的就是`Random Forest Models`

另外bagging的模型可以并行计算，而boosting依赖上一个weak learner,只能串行.


## Boosting

boosting本身不区分回归还是分类问题，都可以使用，看具体用什么样的weak learner + loss函数

### Adaboost

Adaboost是最早的boosting算法，其核心是样本加权，然后针对分错的样本(分类)或者误差较大的样本(回归)，下一个weak learner将重点照顾，直到把错误减小。
而且每个weak learner也都有个权重，最终的模型预测值为:

$$G(x) = sign(f(x))=sign(\sum_{m=1}^{M}(\alpha_{m}*G_{m}(x)))$$


#### About weak learner

问题1:
$$G_{m}(x)$$是第m轮训练的weak learner，具体如何训练，如何选择loss function?

问题2:
上一次分错的，权重越大。如何加权?

[参考理解](https://stackoverflow.com/questions/18054125/how-to-use-weight-when-training-a-weak-learner-for-adaboost)

注意`weak learner`的概念，它是比随机稍微好一点的leaner。而Adaboost里用的weak learner可以是非常简单的单层决策树,先选择特征l，再算分类点a,选择合适的loss，特征分裂后，计算`加权的最小错误分类率(分类)`or `加权的最小平方误差(回归)`,就可以得到本轮的$$G_{m}(x)$$

```
The weak learners in AdaBoost are decision trees with a single split, called decision stumps for their shortness.
```
##### 分类问题

对于分类问题，自然可以用传统分类树里的条件熵，基尼系数这些来当分裂标准(split standard). 

但要注意，最终metric是加权后的错误率，而不是最低的Gini or 条件熵. 而依最低Gini or 熵找到的$$(a,l)$$也许可以得到最低的加权错误率，也许也不可以.总之:

搜索$$(a,l)$$的空间，每次统计按$$(a,l)$$分裂train set后的错误分类率,再加上权重，选择最低$$e_{m}$$的$$(a,l)$$作为$$G_{m}(x)$$.

$$e_{m}=P(G_{m}(x)\neq y_{i})=\sum_{i=1}^{N}w_{m,i}I(G_{m}(x)\neq y_{i})$$

根据$$e_{m}$$更新$$\alpha_{m}$$:

$$\alpha_{m}=0.5* log(\frac{e_{m}}{1-e_{m}})$$

##### 回归问题

对于回归问题，依据mse loss，选好$$(l,a)$$,重点来了，是计算每个样本带加权的rse! 

rse越大，该回归器的权重$$\alpha_{m}$$也越大，错误样本加权的系数w也变大,因此最终得到的minimum rse是朝着最小化错误样本的误差的目标去的.

$$G_{m}(x)=arg\  \underset{l,a}{min}\  w_{i}*\sum_{i=1}^{N}\left ((y(x^{l}_{i})-y)^{2}_{i<a} + (y(x^{l}_{i})-y)^{2}_{i>=a}   \right )$$

进一步，如何根据rse来更新$$G_{m}(x)$$的权重$$\alpha_{m}$$? 

(以下仅是猜测..)

weighted minimun mse应该有一个上界,那就是不加权时的mse!

道理很简单:

$$ax_{1}+bx{2}+c{x_3} < x_{1} + x_{2} + x_{3} , when\  a + b+ c  =1$$

所以有$$r_{m}$$在[0,1]之间:

$$r_{m} = \frac {r_{m, weighted}} {r_{not \ weighted}}$$

再算$$\alpha_{m}$$就可以了:

$$\alpha_{m}=0.5* log(\frac{r_{m}}{1-r_{m}})$$

#### 前向分步加法模型

Adaboost是前向分步加法(stage-wise additive model)的一种,他的特殊在于weak learner是加权的，adaptive嘛.

而一般的Additive Model并没有考虑weak leaner的加权，比如下面提到的boosting tree.

additive model正是Graident boosting预测残差的基础:
$$f_{m}(x)=f_{m-1}(x)+ \alpha * g_{m}$$

## Boosting Tree

以`决策树`作为weak learner的提升方法为提升树, 包括了回归方法和分类方法.

这里的weak learner，比上面adboost用的又要强一点，可能是一颗大树但不完全(有若干结点)，下面以回归问题为例:

最终预测模型仍然是以`分步前向加法模型`为基础:

$$f_{M}(x)=\sum_{m=1}^{M}G(x;\vartheta_{m})$$

$$f_{m}(x)=G(x;\vartheta_{m}) + f_{m-1}(x)$$

第m步前向时:

$$\vartheta_{m}=arg\  \underset{\vartheta_{m}}{min}\sum_{i=1}^{N}L(y_{i},f_{m-1}(x)+T(x_{i},\vartheta_{m}))$$

weak leaner选择回归树，损失函数为`mean square error loss`

$$L=(y-f_{m-1}(x)-G)^{2}=(r-G)^{2},r=y-f_{m-1}(x)$$

r即是上一个分类器与y的残差,第m步的回归器$$G_{m}$$针对该残差r进行估计即可.

求解:

$$f(x)=\sum_{m=1}^{M}C_{m}I(x\epsilon R_{m} )$$

其中$$R_{1},R_{2},...R_{M}$$为回归树划分的M个子区域;

基于平方误差最小(mse)的规则，$$C_{m}$$为各自区域的y的平均值:

$$\hat{C_{m}} =ave( y_{i} | x_{i} \epsilon R_{m}  )$$

![2018-06-20-19-51-34](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-06-20-19-51-34.png)


### Gradient boost

[gradient-boosting-from-scratch](https://medium.com/mlreview/gradient-boosting-from-scratch-1e317ae4587d)

[introduction to gbdt](https://machinelearningmastery.com/gentle-introduction-gradient-boosting-algorithm-machine-learning/)

* 残差拟合

其实上面的残差就是梯度,不过因为loss函数是mse,这个解简单,好算。

而`Gradient boost`方法就是用梯度下降法去解决这一类数值优化问题的方法,loss函数看成是f(x)的函数,
它要求`loss函数对f(x)可导`就行了。

![2018-06-25-07-42-09](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-06-25-07-42-09.png)

```
Generally this approach is called functional gradient descent or gradient descent with functions.
```
[Relative paper link](http://papers.nips.cc/paper/1766-boosting-algorithms-as-gradient-descent.pdf)

对于不是mse的可导的loss函数,用最陡下降法的近似方法，取`loss函数的负梯度在当前模型的取值`来拟合残差，从而估计相应的weak learner:

$$r_{m,i}=-\left [\frac{ \partial L(y_{i},f(x_{i}))}{\partial f(x)_{i}} \right ]_{f(x)=f_{m-1}(x)}$$

```
After calculating the loss, to perform the gradient descent procedure, we must add a tree to the model that reduces the loss (i.e. follow the gradient). We do this by parameterizing the tree, then modify the parameters of the tree and move in the right direction by (reducing the residual loss.

Generally this approach is called functional gradient descent or gradient descent with functions.
```

#### Gradient boost decision tree(GBDT)

GBDT是weak learner 是决策树的Gradient boosting算法

![2018-06-22-10-52-58](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-06-22-10-52-58.png)

其中第c步，是预测leaf的分段regresiion value，如果loss函数为MSE,这个估计就是均值(mean)，如果为MAE,这估计就是中值(median)

#### 如何实现更好的GBDT

GBDT容易overfit

1. Tree Constraints

少几个树模型，叶节点，大一点的特征门限

2. Shrinkage

rmi是求梯度，类似GD,用learning rate控制速率,学的慢，树个数也就多了，需要trade off

3. Random sampling

```
A few variants of stochastic boosting that can be used:
Subsample rows before creating each tree.
Subsample columns before creating each tree
Subsample columns before considering each split.
```
4. Penalized Learning

#### 关于GBDT算法的weak learner的training:

仅用1个特征来简化问题描述

weak learner 此时是一个`desicion tree stumps`, 1个特征,1个切分点,分为R1,R2两个区域:

* 对 r(残差)估计

loss选用L2 loss or MSE(均方误差损失)时,在GBDT里是对 r(残差)估计,求最佳的s:

r包括了幅度和方向

$$s = arg\  \underset{s}{min}\left ( \sum_{x\epsilon R1}(x-c_{1})^{2}+  \sum_{x\epsilon R2}(x-c_{2})^{2}\right )$$

其中

$$c_{1} =  \sum_{y\epsilon R1}y;c_{2} =  \sum_{y\epsilon R1}y$$

实际求解时，可以列出所有的s和c1,c2的组合，可以更直观的看出哪个组合取最小的MSE

* 对 sign(r)即残差的方向估计

另外还有loss function为L1 loss,or MAE (mean absolute error)

MAE的好处是对outlier不会拟合，(中值的缘故，不考虑那些最小最大值),相对MSE,不容易over fit 

在GBDT里是对sign(r)进行估计, 即对仅包括残差方向的vector估计:

$$y-f_{m}(x)=[-1,0,1,1]$$

选择切分点a,并且也sign化，即

$$x<a, \ x=-1;
\\
x=a, \ x=0;
\\
x>a, \ x=1;$$

将样本集x同样变成[1,-1,0,1..]这样的vector，再用`MSE`衡量,求得最佳的a:

这里用`MSE`而不是`MAE`,前者收敛的更快

$$X=[1,1,-1,1]$$

$$a^{*} = arg\ \underset{a}{min}\left (MAE(X,y-f_{m}(x)\right )$$

前面使用`sign(r)`,仅仅是为了更好的划分R1,R2,最终的目的还是要做估计，而不是分类。根据a划分的R1,R2，用median(r)来估计y.

$$\hat{y_{i}} = Median(y \ \epsilon R_{j}), j=1,2,\ i=1,2,...N$$


#### 参考阅读

[Best ever!](http://explained.ai/gradient-boosting)

更详细的解释文中`GBM optimizing MAE by example` 关于L1 loss下 weak learner的预测过程:

![2018-06-22-19-02-12](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-06-22-19-02-12.png)





