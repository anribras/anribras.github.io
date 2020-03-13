---
layout: post
title:
modified:
categories: Tech

tags: [fastai, deep-learning]

comments: true
---

<!-- TOC -->

- [feature engineering](#feature-engineering)
  - [add_datepart](#add_datepart)
  - [统计 row 之间的一个时间特征](#统计-row-之间的一个时间特征)
- [categories 特征和数值特征区分](#categories-特征和数值特征区分)
  - [entity embedding](#entity-embedding)
  - [in fastai?](#in-fastai)
- [神经网络里的回归问题](#神经网络里的回归问题)
- [to summerise](#to-summerise)

<!-- /TOC -->

struct data 就是像 database 里一样的结构化的数据,传统来说 machine learning 里的 data mining 就是干这类事情的

time series 性质的特征如何处理?核心就是下面提到的`add_datepart`以及`get_elapsed`

### feature engineering

如何把不同的表 join 起来.

`store num -> store state->weather and google trends`最后形成一个大的`dataframe`.

#### add_datepart

也属于特征工程里,把一个 date(A)用更详细的多维度的特征来描述:

```
A---->
AYear AMonth AWeek ADay ADayofweek ADayofyear AIs_month_end AIs_month_start AIs_quarter_end AIs_quarter_start AIs_year_end AIs_year_start AElapsed
```

#### 统计 row 之间的一个时间特征

一般新特征都是提取列与列之间的，这个时间特征是在 rows 间提取.

有两个特征是`stateholiday`和`schoolholiday`,与 date 相关，为`１`表示该天是节假日
,通过时间序列(2015-1-1,2015-1-2,2015-1-3...)可以得到一个新特征，比如自从放了`schoolholiday`以来，已经过去了多少天。可以想象到这个特征是和 store 的促销，销量是有直接关系的.

```python
def get_elapsed(fld, pre):
    day1 = np.timedelta64(1, 'D')
    last_date = np.datetime64()
    last_store = 0
    res = []

    for s,v,d in zip(df.Store.values,df[fld].values, df.Date.values):
        if s != last_store:
            last_date = np.datetime64()
            last_store = s
        if v: last_date = d
        res.append(((d-last_date).astype('timedelta64[D]') / day1))
    df[pre+fld] = res
```

### categories 特征和数值特征区分

year,week,dayofweek 被当作`categories`.

未知的统分为一类叫`unkown`

nan 的,如果没有填充，也可以分一类叫 `nan` or `[0,0,0,...0]`.

数值型特征自然每个数值都是不同的度量，甚至带 float 的，和 categories 的还是区别很大.

#### entity embedding

这个主要是针对`categories`特征,区别于`one-hot encoding`

`Entity embedding representation will be equal to nothing but the dot product of one-hot encoded data and learn-able weights.`

why not `one-hot encoding`?

1. 相对维度更低，不容易 overfit,更好计算;
2. 不是稀疏的表达，更准确[01000] vs [0.2 0.9 0.2 0.1 0.2]

[中文理解](http://www.360doc.com/content/18/0305/19/38875511_734538976.shtml)

[参考](https://medium.com/@apiltamang/learning-entity-embeddings-in-one-breath-b35da807b596)

[参考 2](https://machinelearningarchives.blogspot.com/2018/02/entity-embeddings-of-categorical.html)

[参考 3](https://towardsdatascience.com/structured-deep-learning-b8ca4138b848)

[embededing by pytorch](https://github.com/fastai/fastai/blob/master/courses/dl1/lesson5-movielens.ipynb)

其实就是用 deep learning 去做类似结构化特征性质的 data set.

例如,week 是一个 cat 型的特征:

```
week:{"0":"sun","1":"mon","2":"tue","3":"wen","4":"thr","5":"fri","6":"sat"}
```

若按 one-hot encoding，将有 7 个特征维度.

embedding 可以假设其特征展开后只有 4 个维度, {w1,w2,w3,w4},比如某个 sunday 的样本
week=0，初始化为随机的{0.1,0.46,0.88,0.22}，在网络里训练，通过损失函数的梯度反向求导后，最终能找到一个最佳的 w={..}来表示 week=sun,该特征在网络里自己寻找对应 encoding 特征，而不是事先人工指定(one-hot 就是一种特殊的人工指定方法).

deep learning 自己确定特征，不用专家来做，省事. 这种往更高维度的映射(`rich representation`)就是 deep learning on nerual network 的一个重要特点，没法说清楚具体的映射后的数值是什么含义,`玄学`就是这么来的.

像 nlp 里的`word2vec`可以把百万级别的单词特征(词库)映射到只有几百个维度，省事

如果样本数 m 接近某个特征的类别,那这个特征是不适合用`embedding`的

感觉可以把 week 这样的一个特征`embedding`,扩展到 n 个特征上.

```
1 x t --> 1 x s   expands to   n x t ---> 1xs or n x s ? or n x whatever
```

#### in fastai

get_learner，就一个函数...

### 神经网络里的回归问题

前面和分类是一样的，在最后一层的 softmax 修改为 lr 即可，其他 layer 都是一样的

### to summerise

如何用 fastai 架构解决 table structure 类的回归/分类问题

1. 特征工程(可能不需要机器学习那么仔细)
2. 确定 category 和 nnumerical 的特征 put it in dataframe
3. 确定 valadation set idx
4. ColumarModelData.from_data_frame to get model
5. 确定 embedding 后特征的维度
6. ColumarModelData.get_learner 设置基本模型
7. lr_find
8. fix (confirm the metrice)
9. predit

详见`lesson3-rossman.ipynb`

[a good pdf](https://arxiv.org/pdf/1604.06737.pdf)
