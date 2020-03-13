---
layout: post
title:
modified:
categories: Tech

tags: [machine-learning]

comments: true
---

<!-- TOC -->

- [cross-validation](#cross-validation)
- [k-fold](#k-fold)

<!-- /TOC -->

### cross-validation

首先 cross-validation 是用来做`模型选择`的，准确地说，可以是同一 classifier 的不同 hyper parameter 的选择，也可以是不同的 classifier 之间的选择。 cross-validation 的目的不是得到最终的 training model.而是选择一个可能的最佳 model, 在这基础上，再做 full training if necessary.

### k-fold

k-fold cross-validation 的含义是，将 train set 分为 train data 和 validation data, 具体分法就有前面已经知道的`shuffle`,`staightified`等概念. 分成 k 坨，每次用 k-1 坨 train, 剩下的 1 坨打分，因此每个 model 将有 k 个 score，衡量该模型的 final score 可以取 k 个 score 的 mean. 选择最高分作为最优模型。 比如是超参数调优,$\alpha=[0.1,0.001,0.001]$从三个 model 里选择最佳的.

比如 sklearn 里的 gridsearch 的超参数调优，一般可以输入一个 cv:

```python
gsXGB = GridSearchCV(XGBC,param_grid = rf_param_grid, cv=kfold,n_jobs= -1, verbose = 1)
```
