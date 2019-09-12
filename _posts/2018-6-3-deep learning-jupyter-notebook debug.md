---
layout: post
title:
modified:
categories: Tech

tags: [deep-learning]

comments: true
---

<!-- TOC -->

- [问题](#问题)
- [debug](#debug)
- [分析](#分析)
- [解决](#解决)
- [后续！](#后续)

<!-- /TOC -->

### 问题

做线性回归，用到了`mean_squared_log_error`但是提示报错:

```
    Input contains NaN, infinity or a value too large for dtype('float64').
```

### debug

在 jupyter 里，用`%debug`可以很快进入 debug mode

直接在输入框里检查变量值

`up down`可以跳转函数栈.

### 分析

出现了`nan`，是因为 log(pred) 时 pred < 0, 导致 nan 出现.

定位到哪个样本的预测出问题:

```py
    print (np.where(pred <=0 ) )
    pos = np.where(pre d <=0)
    print(len(pos))
    if len(pos) > 0:
        ind = pos[0][0]
        print (pred[ind])
        print (X_vld.iloc[ind])
```

### 解决

基本固定是 2 样本，看来样本有异常，应该剔除。

几方面原因:

1. 有几个样本的 label 值很小，可能特征差不多的情况下，有的又很大，predict 懵逼了

2. 自己在造 feature 的时候，有的 feature 为 nagative(a-b < 0).然后 log skew 时，变成了 nan

3. 可能是模型的问题

最终分析还是还计算`RSME`时出了问题:

出问题的思路,在于没有把 y skew 掉，容易出现 pred <0.

easily error:

```py
pred = predict(X,y); //pred might < 0
rsme =  np.sum (np.square (np.log(pred)-np.log(y) )) / y.shape[0]
```

right way:

```py
y = np.log1p(y)
pred = predict(X,y); //pred got good predictions
rsme =  np.sum (np.square (pred-np.log(y) )) / y.shape[0]
```

有个奇怪的地方是,cv 的选择

```
 #right method, cv = 5
 cross_val_score(LR,X,np.log1p(y),scoring='neg_mean_squared_error',cv=5)

 #not good method, cv 可以选择自己定义的kfolds方法
 cross_val_score(LR,X,y,scoring='neg_mean_squared_log_error',cv=kf)
```

### 后续

还有个问题，data clean 时，采用了`随机初始化na策略`，换个数据，意味着模型又不同了..最好不要用随机初始化 na 的策略
