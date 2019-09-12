---
layout: post
title:
modified:
categories: Tech

tags: [deep-learning]

comments: true
---

<!-- TOC -->

- [A simple start](#A-simple-start)
- [Improving points](#Improving-points)

<!-- /TOC -->

### A simple start

不管是哪个框架，总是要包括这些基础的实现，干脆自己整理一下，有空实现一个

full connection，mini-batch SGD

A Network object

- Network input:

```sh
sizes = [ 5,8,10,2]
X = train_X
y = train_y
optim = SVD(lr = 0.01, batches = 64)
X_test for predict
other
```

- Network basic function:

```sh
init;  input 作为初始参数
forwards(bs_X,bs_y) return cost function 网络结构在这里实现.
backwards() 根据cost function计算梯度 反向传播算法
step()  根据上面计算的梯度，运用SGD更新(w,b)
fit() forwards + backwards + step 得到最终的训练参数 ,使用何种metric cost?
predict() predict for X_test
```

### Improving points

- forwards

```sh
forwards不需要直接算出costfunction,只计算output
可以放到fit里, fit需要指定metric cost
```

- add `epochs`

```sh
整个X重复计算几次?
每次重复计算开始时,是否打乱X顺序? shuffle = True
可以直接用sklearn的各种Kfold ,split来得到，当然自己实现也不难
```

另外 training_nums, epochs 和 batch_sizes 的关系:

如果 batch_sizes 等于 training_nums 即每次输入所有样本，即 full-batch SGD,则不存在 epochs 的概念，每次都需要计算全体样本的 average cost function based on metrice,再求 average graident,更新(w,b),再依据新的(w,b)更新 cost function...

如果 batch_size 比较小，every batch 都更新(w,b)，可能 1 个或几个 epoch, metirices 就已经到达理想值，如果继续计算更多的 epochs,可能导致 over fit 或者已经不起效果.

停止迭代条件可以为 cost function 小于某个值. or 多少个 epochs.

- cross validation

```sh
cross validation应该随着w,b的更新，也给出结果.可以评估当前的训练效果
Using a new function :evaluate(X_cv,y_cv,metrice=m) return metrive result
```

- 中间 metric cost 可视化

```sh
每次迭代更新w,b得到的trainning error和validation error可以输出，便于观察.
Just like what fast-ai did...
```

metric cost 选择: cross-entry , rmse

另外分类 metirces 用 cross-entry, softmax 回归问题用 rsme

- better activation function and metric cost

- Overfitting

- early stopping

这个就是熟悉的交叉验证.train 细分为 3 类，train,validation,和 test,用 train 训练;用 validation 做超参数调优，用 test 评估效果.做交叉验证。随着 trainning 的进行.通过 validation set 的 accruracy 是够饱和,可以判断是否 ovefitting 了. Ng 在 coursera 里讲的很清楚,学习曲线什么的.

这种 validation 超参数调优，又叫`hold out method`

- Weight decay or L2 regularization

- L1 regularization

- Dropout stratgy

```sh
This kind of averaging scheme is often found to be a powerful (though expensive) way of reducing overfitting. The reason is that the different networks may overfit in different ways, and averaging may help eliminate that kind of overfitting.
```

dropout 就是每次训练不同的网络，最后 average 的策略

另外的理解是，每个神经元在自己的上一层可能被 drop 的情况下，其输入是不稳定的，而在这种情况下还能保证模型工作，那么最后的模型是比较稳健(robost)，具有容错(适应更 random 的输入)的

- More training data

1. Weight initialization

```sh
 Then we shall initialize those weights as Gaussian random variables with mean 0 and standard deviation 1/nin‾‾‾√. That is, we'll squash the Gaussians down, making it less likely that our neuron will saturate.

```

不是全部用 randomnorm(0,1)就完事了，针对 l 层，用 randomnorm(0,1/s) ,s = sqrt(num_l),可以让(w,b)更新的更快

更快而已，不会让结果变得更好

- Data augumentation

- Sophastic Graient Desent 的各种版本

已经在 fastai lesson 5 里了解过了

- batch normalazation
