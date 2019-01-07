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

首先cross-validation是用来做`模型选择`的，准确地说，可以是同一classifier的不同hyper parameter的选择，也可以是不同的classifier之间的选择。 cross-validation的目的不是得到最终的 training model.而是选择一个可能的最佳model, 在这基础上，再做full training if necessary.

### k-fold

 k-fold cross-validation的含义是，将train set分为 train data 和 validation data, 具体分法就有前面已经知道的`shuffle`,`staightified`等概念. 分成k坨，每次用k-1坨train, 剩下的1坨打分，因此每个model将有k个score，衡量该模型的final score可以取k个score的mean. 选择最高分作为最优模型。 比如是超参数调优,$\alpha=[0.1,0.001,0.001]$从三个model里选择最佳的.

比如sklearn里的gridsearch的超参数调优，一般可以输入一个cv:
```python
gsXGB = GridSearchCV(XGBC,param_grid = rf_param_grid, cv=kfold,n_jobs= -1, verbose = 1)
```


