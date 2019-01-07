---
layout: post
title:
modified:
categories: Tech
 
tags: [deep-learning]

  
comments: true
---

<!-- TOC -->

- [A simple start](#a-simple-start)
- [Improving points](#improving-points)

<!-- /TOC -->



### A simple start

不管是哪个框架，总是要包括这些基础的实现，干脆自己整理一下，有空实现一个

full connection，mini-batch SGD

A Network object

1. Network input:

```
sizes = [ 5,8,10,2]
X = train_X
y = train_y
optim = SVD(lr = 0.01, batches = 64)
X_test for predict
other 
```


2. Network basic function:

```
init;  input 作为初始参数
forwards(bs_X,bs_y) return cost function 网络结构在这里实现.
backwards() 根据cost function计算梯度 反向传播算法
step()  根据上面计算的梯度，运用SGD更新(w,b)  
fit() forwards + backwards + step 得到最终的训练参数 ,使用何种metric cost?
predict() predict for X_test
```

### Improving points

1. forwards
```
forwards不需要直接算出costfunction,只计算output
可以放到fit里, fit需要指定metric cost
```

2. add `epochs`
```
整个X重复计算几次? 
每次重复计算开始时,是否打乱X顺序? shuffle = True
可以直接用sklearn的各种Kfold ,split来得到，当然自己实现也不难
```

另外training_nums, epochs和batch_sizes的关系:

如果batch_sizes等于training_nums即每次输入所有样本，即full-batch SGD,则不存在epochs的概念，每次都需要计算全体样本的average cost function based on metrice,再求average graident,更新(w,b),再依据新的(w,b)更新cost function...

如果batch_size比较小，every batch都更新(w,b)，可能1个或几个epoch, metirices就已经到达理想值，如果继续计算更多的epochs,可能导致over fit或者已经不起效果.

停止迭代条件可以为cost function小于某个值. or 多少个epochs.

1. cross validation
```
cross validation应该随着w,b的更新，也给出结果.可以评估当前的训练效果
Using a new function :evaluate(X_cv,y_cv,metrice=m) return metrive result
```

4. 中间metric cost可视化
```
每次迭代更新w,b得到的trainning error和validation error可以输出，便于观察.
Just like what fast-ai did...
```

metric cost选择: cross-entry , rmse

另外分类metirces用cross-entry, softmax 回归问题用rsme


5. better activation function and metric cost



6. Overfitting

* early stopping

这个就是熟悉的交叉验证.train细分为3类，train,validation,和test,用train训练;用validation做超参数调优，用test评估效果.做交叉验证。随着trainning的进行.通过validation set 的accruracy是够饱和,可以判断是否ovefitting了. Ng在coursera里讲的很清楚,学习曲线什么的.

这种validation超参数调优，又叫`hold out method`

* Weight decay or L2 regularization

* L1 regularization

* Dropout stratgy

```
This kind of averaging scheme is often found to be a powerful (though expensive) way of reducing overfitting. The reason is that the different networks may overfit in different ways, and averaging may help eliminate that kind of overfitting.
```
dropout就是每次训练不同的网络，最后average的策略

另外的理解是，每个神经元在自己的上一层可能被drop的情况下，其输入是不稳定的，而在这种情况下还能保证模型工作，那么最后的模型是比较稳健(robost)，具有容错(适应更random 的输入)的

* More training data

1. Weight initialization

```
 Then we shall initialize those weights as Gaussian random variables with mean 0 and standard deviation 1/nin‾‾‾√. That is, we'll squash the Gaussians down, making it less likely that our neuron will saturate.

```
不是全部用randomnorm(0,1)就完事了，针对l层，用randomnorm(0,1/s) ,s = sqrt(num_l),可以让(w,b)更新的更快


更快而已，不会让结果变得更好

8. Data augumentation

9. Sophastic Graient Desent 的各种版本

已经在fastai lesson 5里了解过了

10. batch normalazation

